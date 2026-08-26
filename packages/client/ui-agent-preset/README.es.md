# dsh-client-ui-agent-preset

[English](README.md) | [中文](README.zh.md) | Español

Las superficies de agent-preset: una fila en los ajustes Generales que elige con qué [preset](../../preset/agent-presets/README.es.md) se componen las sesiones nuevas, un chip en la pantalla de nueva sesión que elige el de la siguiente, una etiqueta de solo lectura en la cabecera de sesión, y una sección de ajustes que gestiona el roster — copiar, eliminar, el predeterminado y el camino a los propios archivos del preset.

## Por qué es una preferencia de nueva sesión

El preset de una sesión queda fijado cuando la sesión se crea — el host se niega a adoptar una sesión existente bajo otro distinto, porque el historial de esa sesión se produjo con las herramientas del primer preset. Así que esta fila no puede ser un conmutador en vivo, y lo dice: cambiarlo se aplica a las sesiones iniciadas después, mientras que las sesiones en ejecución conservan la composición con la que empezaron.

## El chip de nueva sesión

Una segunda superficie, junto al selector de workspace de la pantalla de nueva sesión. Está ahí en lugar de en el composer porque es donde la elección sigue abierta: un control que pasa la mayor parte de su vida deshabilitado pertenece a la pantalla donde todavía funciona.

El chip se abre con el predeterminado del despliegue y su elección queda *en escenario (staged)* — la pantalla precede a la sesión a la que se aplicaría. El escenario alcanza a una sesión cuando una se vuelve actual y sigue en blanco, lo que cubre tanto la sesión que creó el connect de workspace como la en blanco que reutilizó; montarse en `sessions.create` se perdería la segunda. Se gasta en el primer uso, de modo que la siguiente sesión nueva vuelve a abrirse con el predeterminado, exactamente igual que el selector de workspace de al lado.

Una sesión que ya ha empezado se rechaza en lugar de ponerse en cola: el host responde `agent-preset-locked` y el escenario se descarta en lugar de esperar a una sesión que nunca lo aceptará.

## La etiqueta de la cabecera de sesión

Una tercera superficie, junto al título de la sesión: el preset con el que corre ESTA sesión, como chrome estático. Un control allí prometería un conmutador que el host rechaza de plano. Lee el preset del propio resumen de la sesión y resuelve el nombre visible contra el mismo roster que lee la fila de Generales. Los eventos de dueño `agent-preset/selected` reenviados pliegan los cambios confirmados de sesión en blanco en ese resumen compartido en todas las pestañas; la pestaña iniciadora puede que ya haya aplicado el eco del RPC, y la fusión es idempotente.

## Qué lee y qué escribe

Las opciones y el predeterminado actual provienen ambos de una sola llamada `agentPreset.list`. El roster ya reporta qué id recibe una sesión sin elección explícita, así que la fila no necesita introspección del schema de ajustes; la escritura apunta al campo `default` del namespace de ajustes `agent-presets`, que es lo que el host resuelve en la creación.

Un preset creado localmente tiene exactamente los mismos privilegios que los plugins que nombra, así que la lista marca las filas `user` en lugar de presentar cada preset como incluido de serie y auditado.

Los archivos de preset publican un `name` y una `description` sin localizar, que Web usa para cada fila `user` y para las filas `system` desconocidas. Para los cuatro ids incluidos de serie (`standard`, `code`, `minimal` y `cordis`), Web resuelve ambos campos desde su locale activo solo cuando el roster marca la fila como `system`; un preset `user` con el mismo nombre conserva los metadatos de su archivo.

La fila se vuelve a leer ante `settings/changed` de su propio namespace y ante `connection/reset`: el roster es un directorio en vivo y el predeterminado es un campo de ajustes, así que tanto una edición externa como una reconexión pueden moverlo.

## La sección de gestión

Una cuarta superficie, su propia página de ajustes (id de `settings.section` `agent-presets`, ordenada después de Models — elegir un modelo es rutinario, componer un agent (agente) es el acto que da forma al despliegue que hay detrás): el roster como tarjetas, un diálogo de copia como única forma de crear un preset, y un visor de solo lectura sobre las composiciones incluidas.

El navegador no edita texto de composición. Editar YAML en un textarea web era una superficie débil (sin autocompletado, sin resaltado, sin diff), así que un preset nuevo es una copia del lado del host de uno existente — el diálogo recoge un id (se convierte en el nombre del directorio, por eso debe nombrarse de antemano y no puede cambiarse después) y un nombre visible opcional, y `{ from, id, name? }` es todo lo que cruza el cable. Todo lo demás — descripción, composición, skills (destreza) — se edita en los propios archivos del preset, y el otro trabajo de la página es llevar al usuario A esos archivos: la copia se completa abriendo el nuevo directorio, y cada fila personalizada conserva una acción de ubicación. Cuando el host no tiene abridor de escritorio (`hasDocument: false` en el roster; despliegues remotos y en contenedores), las mismas acciones responden el directorio como texto en la fila en lugar de ofrecer un botón que haría spawn en el vacío.

Un preset publica su propia descripción, de cualquier longitud, y la cuadrícula dimensiona todas las filas de tarjetas por igual — así que una descripción sin límite fijaría la altura de todo el roster. Las tarjetas la limitan a cuatro líneas y ofrecen el resto en un tooltip, adjunto solo mientras el texto está realmente cortado. El recorte es CSS, de modo que la descripción completa permanece en el árbol de accesibilidad sea lo que sea lo que muestre la tarjeta.

Un preset incluido de serie se abre en el visor de solo lectura. Es la composición conocida y correcta desde la que parte una copia, así que leerla es el objetivo; no ofrece ubicación ni eliminar — su instalación la sobrescriben las actualizaciones y no es del usuario gestionarla. La introducción transporta la orientación que un botón de crear solía implicar: duplica un preset existente y hazlo tuyo, o deja que el agent redacte uno en modo Creator.

Junto a copiar está la entrada conversacional: cuando el roster transporta el preset autorreferencial `cordis`, una tarjeta de añadir punteada (el affordance de la página de Models) lo pone en escenario e inicia una nueva sesión — la sección cierra el panel de ajustes mediante el owner-prop `close` del shell y el propio aplicador del chip de nueva sesión compone la sesión en blanco que produce el flujo de workspace. El asiento evita que una carga tardía del roster regrese la pantalla: primero la elección en escenario, luego la composición que la sesión actual ya transporta, y después el predeterminado del despliegue.

El diálogo refleja la propia regla de contención del host (`[a-z0-9][a-z0-9-]*`) y rechaza un nombre ya en uso — una copia nunca sobrescribe. Ambas comprobaciones son comodidades: el host las vuelve a aplicar y su respuesta es lo que el diálogo reporta en caso de fallo.

Eliminar quita el directorio del preset. Las sesiones ya compuestas desde él siguen ejecutándose — una composición se monta una vez en la creación de la sesión y nada vuelve a leer el archivo.

Una fila del roster que transporta `broken` (la comprobación de forma del host encontró la composición ausente o no cargable) se renderiza como tarjeta marcada: borde rojo, una insignia de "Failed to load" (lo que observó el descubrimiento, no una afirmación de que los archivos estén dañados — la causa habitual es una composición que el usuario acaba de editar o eliminar), la razón verbatim, el cuerpo deshabilitado — no puede convertirse en el predeterminado — y la duplicación deshabilitada, porque una copia de un preset roto es otro preset roto. Una fila personalizada rota conserva sus acciones de ubicación y eliminación, porque los archivos son donde se arregla y eliminar es como se limpia un directorio fantasma (composición eliminada a mano, directorio bloqueando aún el id); una fila incluida rota retiene también el visor — no hay composición legible que mostrar. Los dos selectores (la fila de Generales y el chip de nueva sesión) descartan por completo los presets rotos: eligen la composición de la SIGUIENTE sesión, y ofrecer uno que no puede componer solo diferiría el fallo al inicio de la sesión.

Fijar el predeterminado escribe el namespace de ajustes `agent-presets`, que el host expone a los clientes de configuración ([`dsh-apiproxy`](../../host/apiproxy/README.es.md) mantiene una allowlist explícita — un namespace fuera de ella hace que un selector se mueva y luego olvide en silencio).

`agentPreset.read`, `copy`, `openDocument` y `remove` están fijados al loopback ([`dsh-client-connection`](../connection/README.es.md)): una composición nombra los plugins que ejecuta una sesión, así que leer una es reconocimiento, y el resto gestiona el roster y conduce el escritorio del host. `agentPreset.list` no lo está — transporta ids, confianza y las dos banderas de capacidad sin rutas, y el selector de un cliente LAN lo necesita.

## Cuando las superficies están ausentes

Un despliegue que no compone presets responde con un roster vacío, y la fila, el chip, la etiqueta y la sección no renderizan nada — cada sesión comparte entonces la composición del host, y no hay nada entre lo que elegir ni nada que gestionar. Un despliegue que no configura ninguna raíz escribible responde `authorable: false`, y la sección sigue siendo un navegador de solo lectura: las composiciones incluidas siguen abriéndose en el visor, pero cada acción de copia está deshabilitada con la razón como tooltip en lugar de ofrecer un diálogo cuyo crear falla siempre.

## Experiencia de modelo

Indirectamente, a través del preset desde el que se compone una sesión posterior; [`dsh-agent-presets`](../../preset/agent-presets/README.es.md) es el dueño de lo que esa composición pone delante del modelo.

#### Efecto en la caché KV

Ninguna invalidación directa. Cambiar el predeterminado nunca toca el prefijo de una sesión en ejecución; una sesión creada después establece su propio prefijo desde su propia composición.

## Limitaciones conocidas y trabajo pendiente

- **Un preset sin metadatos se lista por id** — el texto visible es opcional, y una copia sin nombre recurre deliberadamente a su nombre de directorio en lugar de presentarse idéntica a su origen.
- **Una ruta revelada es texto visible, no un enlace** — donde el host no tiene abridor de escritorio, la fila muestra el directorio para copiarlo a mano; el navegador no puede abrir por sí mismo una ubicación del sistema de archivos del host.
- **Las ediciones de composición son invisibles para la página** — los archivos se editan fuera del navegador y nada en el cable anuncia un cambio de archivo, así que el roster se vuelve a leer con sus propias acciones, `settings/changed` y `connection/reset`, no con cada edición en disco.
