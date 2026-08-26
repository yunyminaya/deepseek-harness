# Agent Note: ACP como protocolo solo de automatización

Status: implemented

[English](2026-07-23-acp-automation-only-protocol.md) | Español

## Problema

El bridge de ACP se había convertido en una segunda UI de producto interactiva. Traducía eventos durables en tarjetas de editor, metadatos de terminal, diffs, planes, títulos, razonamiento, comandos, modos, selectores de modelo y permiso, navegación de sesiones y elicitación humana. Esas responsabilidades duplicaban la TUI y el cliente Web mientras acoplaban un transporte de automatización a servicios de UI, consultas de persistencia, política de presentación y convenciones específicas del editor.

ACP aún tiene un papel útil: otro agente o controlador automatizado puede arrancar un proceso del harness, crear una sesión aislada, enviar texto o una imagen en línea estrechamente soportada, recibir la respuesta de texto/imagen confirmada, cancelar trabajo y responder a una solicitud de permiso. El backend de subagentes ACP fuera de proceso depende de ese límite de protocolo estándar.

La suite de instantáneas complica la eliminación. La mayoría de los escenarios ACP ejercitan el backend de agente ensamblado en lugar de la presentación ACP, así que eliminar la suite junto con el bridge de editor descartaría una cobertura conductual amplia sin clave.

## Decisión

`@deepseek-ai/dsh-acp` es un transporte de automatización bajo [`packages/acp/acp`](../../../../packages/acp/acp/README.md), fuera del grupo de paquetes `ui`. Su protocolo público es intencionadamente pequeño: negociación de versión, sesiones nuevas con un prompt en vuelo cada una, actualizaciones de texto/imagen de asistente confirmadas, cancelación por sesión, sesiones concurrentes y teardown propiedad de la conexión. Los prompts preservan texto e imágenes rasterizadas soportadas en orden de cable, mientras que los enlaces de recursos se aplanan a referencias textuales entre corchetes; el bridge rechaza directorios adicionales, servidores MCP, audio, recursos incrustados, prompts malformados o vacíos, sesiones desconocidas y prompts solapados.

La capacidad de imagen es veraz en lugar de estructural: `initialize` la anuncia solo cuando existe un almacén de adjuntos durable y el provider/modelo exacto configurado resuelve con entrada de imagen explícita. Cada prompt de imagen re-comprueba la ruta exacta más reciente de la sesión, decodifica estrictamente cada bloque y delega el lote completo a `AttachmentStore.saveImages()` antes de publicar el evento de usuario. La cancelación reserva y aborta el slot de admisión antes de cualquier trabajo asíncrono, espera a que las escrituras ya iniciadas se calmen antes de que el prompt se asiente, y nunca publica un mensaje tardío; antes de que el prompt entre en el inbox del Agent no cancela ni espera trabajo de Agent no relacionado. Una escritura completada con dirección por contenido puede quedar inalcanzable porque el rollback destructivo no es válido para un almacén deduplicado. Los fallos de política de imagen corregibles por el llamador se mapean a parámetros inválidos, mientras que la búsqueda de ruta, la corrupción de almacenamiento y los fallos de persistencia siguen siendo faltas internas.

El bridge emite solo texto e imágenes confirmadas de `assistant/message`. Una cadena de promesas por sesión preserva el orden de bloques y mensajes mientras las referencias de imagen de asistente se releen asíncronamente y se verifica su integridad para la entrega base64 de ACP; un objeto ausente o corrupto falla la entrega del prompt en lugar de convertirse en un placeholder. El razonamiento, los chunks crudos, la actividad de herramientas, los todos, los planes, los títulos, los marcadores de retry, los metadatos de terminal, los diffs, las ubicaciones y los enlaces de recursos permanecen en el log de sesión durable o en transportes específicos de UI. No proporciona carga/listado/borrado de sesiones, comandos, modos, selectores de configuración, cambio de modelo, revisión de planes ni elicitación humana.

`session/request_permission` one-shot permanece. Es un canal de política de máquina para agentes propiedad del bridge, no una UI de aprobación humana: el answerer acepta solo un objeto de agente exacto en el mapa de sesiones en vivo del bridge, delega las solicitudes ajenas o sin llamada, y mapea los RPC fallidos al resultado no disponible de fail-closed. El cliente elige allow una vez, reject una vez o cancel, y el bridge nunca convierte esa respuesta en una concesión durable. La política de preguntar permanece en el seam de aprobación y sus productores; [`dsh-subagent-acp`](../../../../packages/subagent/subagent-acp/README.md) usa este canal programáticamente.

La composición de la app contiene el spine de agente, la persistencia, la política de checkpoint y el transporte ACP. No monta servicios de comando, session-query, session-reference, plan-mode, permission-picker ni user-questions para ACP.

El transporte programa servicios de agente, sesión y aprobación a nivel de interfaz en lugar del agent loop concreto. La ejecución de herramientas permanece dentro del harness; ACP nunca delega la ejecución de shell a un editor. El stdout lleva solo JSON-RPC con marco, así que la app no monta ningún logger de stdout y el bridge no hace monkey-patch de la salida del proceso.

La desconexión y la destrucción del plugin comparten un único límite de quietud memoizado. Tanto el cierre de transporte exitoso como el fallido cancelan la admisión de prompts y los agentes, drenan la salida ordenada, asientan los prompts pendientes como cancelados, destruyen cada agente propiedad del bridge y esperan la limpieza del loop y de la sesión. Un create que pierde la carrera de cierre destruye su handle no publicado.

## Límite de instantáneas

La suite de instantáneas ACP sigue arrancando el ejemplo ACP ensamblado y conserva los escenarios que fijan el comportamiento del backend. Solo los escenarios conducidos por métodos de UI eliminados salen de la suite; la recuperación de checkpoint semántico se ejecuta a través del ejemplo headless `stream-json` porque ACP ya no carga sesiones.

Los tests de protocolo y ciclo de vida fijan los codecs de stop-reason, la negociación de versión, la capacidad de imagen veraz, la creación de sesiones nuevas, la admisión ordenada de texto/imagen, el aplanamiento de enlaces de recursos, la validación de todos los miembros antes de las escrituras, la ausencia de base64 en línea en los eventos durables, el rechazo de prompts vacíos o no soportados, la propiedad exacta de permiso del agente, el aislamiento multi-sesión, el asentamiento de prompts después de la salida ordenada, la entrega verificada de imágenes de asistente, la cancelación durante la admisión sin followup tardío ni cancelación de trabajo de Agent no relacionado, la exclusión de fallos pre-inbox no relacionados, el cierre de transporte fallido, la limpieza de recarga solo-ACP y la quietud de teardown. Una instantánea sin clave ensamblada envía un PNG en línea real a través del ejemplo ACP ejecutable y fija solo su referencia durable en el log de sesión. Los smokes compilados y de stdio real rechazan stdout extraviado. La rama `session/new` que pierde una carrera de cierre de stdio real sigue exenta de cobertura porque el transporte en memoria no puede reproducir ese orden; destruye el handle no publicado, mientras que los tests de destrucción circundantes fijan el invariante sin-huérfanos.

## Alternativas consideradas

**Conservar ACP como UI de editor hasta que Web alcance la paridad.** Rechazado porque deja dos contratos interactivos que evolucionan y mantiene convenciones de editor en el límite de automatización.

**Conservar el bridge de editor anterior detrás de límites de servicio disciplinados.** Rechazado aunque ese bridge usara correctamente servicios de interfaz, intenciones de render propiedad de herramientas, answerers de aprobación y user-questions, ejecución propiedad del harness y una composición de stdout puro. Sus tarjetas de terminal eran proyecciones `_meta` de Zed solo-visuales y con puerta de capacidad con un fallback de texto, no `terminal/create` de ACP, así que la ejecución de shell nunca salía del harness. La proyección derivaba cada id de terminal de visualización del id estable por llamada para prevenir colisiones y recuperaba el código de salida o la señal de los marcadores de estado renderizados porque el presentador de resultados puro recibía bloques de contenido en lugar de una salida estructurada; los tests de ida y vuelta de marcadores y un test explícito de fallback `console` sin capacidad fijaban ambos contratos. Esos límites eran coherentes pero no podían hacer que las tarjetas de editor, la navegación de sesiones, los selectores de configuración y la elicitación humana pertenecieran a un protocolo de automatización.

**Reemplazar ACP por un RPC privado de subagentes.** Rechazado porque ACP ya suministra un protocolo de proceso tipado e interoperable y lo usa el backend de subagentes fuera de proceso.

**Eliminar las solicitudes de permiso de máquina con las demás funcionalidades de interacción.** Rechazado porque un padre automatizado debe responder a la decisión de política one-shot de un agente hijo; esto es flujo de control entre agentes, no presentación.

**Eliminar la suite de instantáneas ACP o migrar cada escenario en este cambio.** Rechazado porque la mayoría de los escenarios prueban el backend y siguen siendo valiosos, mientras que una migración completa del harness es un cambio de testing independiente. Solo los escenarios cuyo driver era un método de UI eliminado salen de esta suite.

**Anunciar soporte de imagen siempre que el SDK de ACP tenga un bloque de imagen.** Rechazado porque el vocabulario del protocolo no prueba que este despliegue pueda persistir bytes ni que la ruta exacta configurada acepte entrada visual. La capacidad desconocida es falsa en la inicialización; la admisión de prompts re-comprueba la ruta en vivo.

**Aplanar las imágenes en línea y de asistente a marcadores o persistir el base64 de ACP en los eventos de sesión.** Rechazado porque los marcadores pierden silenciosamente la intención de modelo/usuario y el base64 convierte los logs durables en el almacén binario. ACP traduce entre su bloque de cable y la referencia `ImageBlock` durable existente en el límite del transporte.

**Crear un servicio RichContent genérico para ACP, MCP y Web.** Rechazado porque `ContentBlock` de core más el seam de adjuntos ya son propietarios del contrato compartido. Cada puerta frontal conserva solo el parseo de protocolo, la prueba de capacidad y la orquestación de ciclo de vida; los límites de lote compartidos y la validación de imágenes permanecen en `AttachmentStore.saveImages()`.

## Consecuencias

ACP tiene un contrato estrecho adecuado para agentes y automatización, mientras que TUI y Web son propietarios de la interacción humana y la presentación. El paquete tiene menos servicios inyectados, dependencias, ramas de protocolo y estados de ciclo de vida, y ya no reclama compatibilidad como punto de entrada de editor general.

Los clientes de automatización reciben texto/imágenes confirmados completos en lugar de deltas de tokens o UI de herramientas estructurada. Inspeccionan logs durables u otra API cuando necesitan razonamiento, trazas de herramientas, títulos o estado más rico. La operación solo-sesiones-nuevas también significa que los llamadores que necesitan navegación durable o reanudación usan una API de host en lugar de ACP.

La cobertura de instantáneas del backend por tanto sigue acoplada al transporte ACP aunque ese transporte sea incidental al comportamiento bajo prueba.
