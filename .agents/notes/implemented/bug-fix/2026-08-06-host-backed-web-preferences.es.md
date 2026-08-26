# Agent Note: Persistir las preferencias de usuario de Web a través de los ajustes del Host

[English](2026-08-06-host-backed-web-preferences.md) | Español

Estado: implementado

## Problema

Las preferencias de Web Appearance, Language y busy-Enter vivían en el `localStorage` del navegador. El almacenamiento del navegador está limitado a un origen, así que reabrir `dsh web` en otro puerto seleccionaba una partición distinta y perdía las elecciones, aunque ambos procesos usaran el mismo home de DSH. Son preferencias de producto a nivel de usuario; la selección de sesión, los borradores, el estado de despliegue de paneles y demás estado transitorio del navegador siguen siendo locales a la página.

La primera implementación del tema movió solo Appearance a los ajustes del Host, pero esperaba su RPC inicial antes de proporcionar `ThemeRuntime`. Una petición de ajustes lenta o no disponible suspendía por tanto la página ensamblada. Además, se suscribía después de la lectura, podía perder una invalidación en esa ventana, no llevaba las revisiones del espacio de nombres en las escrituras y permitía que escrituras en cola de un plugin destruido llegaran al Host.

## Decisión

Las mitades del Host propietarias registran tres schemas: `locale.preference` opcional (`zh` o `en`, donde la ausencia delega en el navegador), `ui-theme.preference` (`light`, `dark` o `system`, por defecto `system`) y `ui-conversation.busyEnter` (`queue` o `steer`, por defecto `queue`). El provider de ajustes local almacena las elecciones explícitas en `$DSH_HOME/settings.yaml`, que se resuelve a `~/.dsh/settings.yaml` con el home por defecto. El proxy de API sirve cada espacio de nombres registrado a un cliente de loopback; los roles de campo siguen ocultando los secretos.

`dsh-client-ui-settings` es propietario de un único espejo describe de ajustes para todo el navegador y proporciona `ctx.settingsScope.bind(spec)` como selector por espacio de nombres sobre él. El espejo instala los listeners de `settings/document-updated` y `connection/reset` antes de iniciar su lectura en segundo plano, así que ningún transporte de ajustes puede bloquear la activación del plugin y una invalidación no puede caer en un hueco de lectura-antes-de-suscripción. Cada scope enlazado publica un store de instantáneas (estado, valor de sección, revisión, capacidad de escritura, modo host/memoria) al que se suscribe el servicio de dominio, sin añadir una lectura de transporte ni un listener propios. El decodificador por defecto valida cada sección entrante contra el schema de transporte serializado del propio espacio de nombres, rehidratado a través del servicio co-ubicado `ctx.settingsSchema`, así que los dominios no llevan protecciones de transporte escritas a mano. Los servicios de dominio reciben el scope como colaborador ordinario del constructor, publican de inmediato sus valores provisionales por defecto —locale derivado del navegador, tema del sistema y Queue— y después adoptan una sección del Host aceptada sin escribirla de vuelta; un servicio construido sin scope (diccionario independiente o fixtures de política) simplemente permanece local al proceso. El ciclo de vida compartido de lectura e invalidación lo especifica la posterior [decisión del espejo describe de ajustes](../architecture/2026-08-17-settings-describe-mirror.es.md).

Los cambios del usuario actualizan el servicio vivo de forma síncrona y ponen en cola una operación de la ruta `settings.mutate` a través de `scope.set`. El scope serializa los gestos, envía la última revisión conocida del espacio de nombres como `expectedRevision`, registra cada revisión con éxito y deja que solo la resolución de la escritura más reciente vuelva a publicar el estado vivo. Una escritura más reciente rechazada o fallida recarga el estado del Host. La destrucción rechaza trabajo nuevo, omite las operaciones en cola, suprime la publicación de la operación en curso y espera a que esa operación se resuelva antes de que el plugin alcance la quiescencia.

Los navegadores remotos no pueden llamar a la API de configuración solo-loopback, así que sus preferencias siguen siendo locales al proceso. Los ids de tema dinámicos de terceros siguen siendo extensiones en proceso fuera del schema integrado del Host; eliminar uno reinicia el registro vivo sin reemplazar la última preferencia integrada persistente.

## Alternativas consideradas

**Mantener `localStorage` y copiar los valores entre puertos.** Un origen no puede enumerar el almacenamiento de otro origen, y un relay del Host recrearía el servicio de ajustes alrededor de un formato específico del navegador.

**Reflejar los ajustes del Host en `localStorage`.** Una segunda autoridad exigiría reglas de conflicto de arranque e invalidación conservando la partición que causó el defecto. El documento del Host es la única fuente persistente.

**Esperar a la lectura inicial para evitar un renderizado provisional.** La disponibilidad de configuración no es un requisito previo para dibujar la página. Una lectura en segundo plano puede causar una convergencia en vivo, pero mantiene el fallo aislado y conserva los respaldos existentes de navegador/sistema/por defecto.

**Dar a cada dominio su propio controlador de ajustes.** Las reglas de concurrencia, revisión, fallo, invalidación y destrucción son idénticas; copiarlas ya produjo deriva de ciclo de vida en la implementación del tema. Los schemas propiedad del dominio mantienen la política de producto fuera del runtime compartido.

**Un controlador de preferencias por campo con callbacks emparejados de sync/persist.** El primer ciclo de vida compartido sincronizaba un campo escalar mediante un callback de `sync` del dominio mientras el servicio escribía de vuelta a través de un callback de `persist` inyectado. Los callbacks mutuos forzaban una construcción en dos fases —un escritor sin operación por defecto reemplazado después mediante `bindPersistence`—, cada campo adicional de un espacio de nombres habría llevado su propio controlador y una lectura de todo el documento, y cada dominio re-declaraba una protección escrita a mano que el schema de transporte registrado ya expresa. El scope de espacio de nombres publica una instantánea a la que el servicio se suscribe y acepta escrituras directamente, así que el par de callbacks y la segunda fase de construcción no existen.

**Mover cada entrada de `localStorage` a los ajustes.** La sesión actual, los borradores, el despliegue de paneles, el estado de visualización de la trayectoria y demás entradas son estado de la instancia del navegador, no configuración del usuario. Promocionarlas sincronizaría estado de navegación transitorio entre pestañas y puertos sin un contrato de producto.

## Consecuencias

Las elecciones de Appearance, Language y busy-Enter siguen al home de usuario de DSH entre recargas, puertos y orígenes de loopback. Las ediciones directas de `settings.yaml` convergen a través del flujo de invalidación existente, mientras que las entradas heredadas `dsh.theme`, `dsh.locale` y `dsh.conversation.busyEnter` no se leen ni se escriben.

El arranque puede mostrar brevemente el valor por defecto del dominio antes de que la lectura en segundo plano se resuelva. Un fallo de lectura transitorio conserva ese valor por defecto o el último valor válido en proceso; la reconexión reintenta. Un rechazo de escritura puede restaurar visiblemente la preferencia persistente después del cambio local inmediato.

La cobertura unitaria enfocada fija el registro de schemas, el orden listener-antes-de-lectura, la activación no bloqueante, la aceptación de secciones validadas por schema, las escrituras ordenadas por revisión, la contención de respuestas obsoletas, la recuperación ante fallos, la quiescencia de destrucción y el modo de memoria remoto. El scope con granularidad de espacio de nombres también admite secciones multi-campo, así que las futuras superficies de configuración pueden montarse en el mismo ciclo de vida en lugar de implementar a mano la sincronización describe/mutate. El escenario Web de ajustes sin clave escribe las tres preferencias a través de la interfaz, verifica el documento YAML y el almacenamiento heredado vacío, recarga y arranca otro Host en un puerto distinto contra el mismo home de DSH.
