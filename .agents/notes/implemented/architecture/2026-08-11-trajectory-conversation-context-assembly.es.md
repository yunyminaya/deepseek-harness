# Agent Note: Ensamblado de Trajectory a partir de Conversation Contexts registrados

Status: implemented

[English](2026-08-11-trajectory-conversation-context-assembly.md) | [中文](2026-08-11-trajectory-conversation-context-assembly.zh.md) | Español

## Problema

Trajectory mantenía una fuente independiente de historial de Session y plegaba la ventana completa de Events cargada en estado de Assistant, Tool, mensaje, Request-header y Compaction. Chat ya ensamblaba las mismas familias de eventos a través de Conversation Definitions registradas. Las dos rutas duplicaban la correlación de negocio y el comportamiento de paginación, y una actualización estructural de Trajectory copiaba o re-escaneaba trabajo proporcional al número crudo de Events aunque cambiara un solo objeto de negocio.

Reutilizar los Nodes finales de Chat no resolvería el problema de propiedad. Trajectory necesita ciclos de vida de request, estado de Assistant en ejecución, herencia de prompt, schemas de Tool, registros de tiempo y un modelo de lectura orientado a etapas que Chat no consume. Compartir los payloads de Node finales acoplaría ambas vistas a la unión de sus requisitos.

La migración también tenía que preservar la clasificación durable de steering. Un `user/message` no dice si abrió un Turn o fue reclamado de la bandeja `next-step`, y una página más antigua puede suministrar el predecesor de bandeja o Location que falta después de que el mensaje ya se haya materializado.

## Decisión

Trajectory registra Conversation Definitions propiedad del objetivo y un View Builder `trajectory` contra el [`ConversationNodeAssembler`](2026-08-09-client-conversation-node-assembly.es.md) compartido. Session es dueña de una ventana de Events contigua y publica instantáneas de Chat y de Trajectory a través de `Session.views`; no ejecuta una segunda fuente de historial de Trajectory ni un pliegue de negocio.

Cada Definition pertenece a un objetivo. Chat y Trajectory pueden reconocer la misma familia de Events durables, pero mantienen State y payloads de Node finales separados. Comparten solo el emparejado por ID exacto del Assembler, los Matches ordenados, los hechos de Location, las dependencias de Reader, la programación de publicación y el ciclo de vida de replace/prepend/append.

El [registro de inspección de Trajectory](../feature/2026-07-27-trajectory-inspection-ledger.es.md) existente sigue siendo el modelo de vista. El Builder de Trajectory convierte los Nodes objetivo materializados en sus `eventNodes`, Requests, schemas de Tool, llamadas en ejecución y mapa de Location establecidos; el layout, la virtualización de tablas, la selección, la vista de Overview y el comportamiento del inspector no se convierten en contratos genéricos de Conversation.

### Business Definitions

| Business | Identidad de Context | Ensamblado de State | Contribución de Trajectory |
|---|---|---|---|
| Bandeja `next-step` | seq de Event de splice | Aplica el splice al Context de bandeja precedente más cercano | Solo State; sin Node visible |
| Mensaje de usuario, steering o inyectado | seq de Event de mensaje | Lee el State de bandeja precedente y clasifica el mensaje durable | Node de entrada o de contexto |
| Assistant y Request ordinarios | `turn:step` | Pliega `step/start`, chunks, mensaje final, retry y `step/end` | Assistant final, Assistant parcial y Request |
| Llamada de Tool raíz | ID de llamada raíz | Pliega la llamada/resultado raíz y los eventos anidados de Code Dispatch en un árbol de llamadas | Árbol de Tool final o en ejecución |
| Compaction | ID de compactación | Pliega start, summary, end y el checkpoint de reemplazo | Request de Compaction |
| Request header | seq de Event de cabecera | Lee la cabecera precedente y retiene el prompt efectivo más el cambio real | Fuente de Prompt y schema de Tool |
| Límites de Session y Turn | seq de Event de límite | Retiene el tiempo de cierre y los hechos de error | Compaction interrumpida o Request ordinario fallido |

Todo Event que correlacione debe exponer directamente el mismo ID de negocio. Code Dispatch usa `rootCallId`, Compaction usa su ID de compactación, y los eventos ordinarios de Tool y retry conservan sus identidades de protocolo incluso cuando una Definition concreta correlaciona por `turn:step`. Los registros heredados que carecen del ID de correlación requerido son ignorados por esa Definition en lugar de fusionarse en un Context `undefined` o tumbar la Session.

Los chunks de Assistant actualizan solo su Context de `turn:step`. Los chunks con contenido solicitan publicación en frame de animation; los chunks de uso y finalización actualizan el State sin forzar su propio frame. Un mensaje final, retry o límite publica inmediatamente. El State de Assistant completado retiene los bloques ensamblados, el tiempo, el uso y los hechos de retry en lugar de copiar el registro crudo de chunks a la instantánea objetivo.

### Steering desde Contexts predecesores

Trajectory reconstruye el steering desde el historial durable de bandeja, usando la misma regla de identidad que la [decisión de steering de Chat](../feature/2026-08-04-web-context-source-and-steer-marks.es.md) sin compartir el Node final de Chat.

Cada Event `agent/inbox/spliced` dirigido a `next-step` inicia un Context invisible identificado por su seq de Event. Su `start()` lee el Context de bandeja anterior más cercano, aplica el splice y almacena las identidades pendientes más el conjunto acumulado de IDs de mensaje reclamados. Un `user/message` posterior de origen usuario lee el Context de bandeja anterior más cercano: un ID reclamado produce un Node de Steering, mientras que cualquier otro mensaje de origen usuario produce un Node de Usuario ordinario.

Una miss del Reader mientras queda historia más antigua registra una dependencia de hueco de ventana. Cuando el prepend suministra el predecesor ausente, el Assembler reproduce la cadena de bandeja afectada y los Contexts de mensaje en orden de Event hacia delante. La dirección de paginación histórica no puede, por tanto, clasificar mal un mensaje de forma permanente.

El Location del Event de mensaje coloca el steering en el Step dueño. Si la ventana de historial cargada no tiene suficientes Events de límite para resolver ese Location, el layout usa el Step de Assistant siguiente como fallback posicional. Un marcador de Request en ejecución sigue a la entrada de steering inicial en el mismo Step, de modo que el marcador denota el Request de modelo causado por esa entrada en lugar de aparecer antes que ella.

### Rutas de ventana y complejidad

Sea `E` el número crudo de Events cargados, `P` una página recién prepended, `D` el número de Definitions de Trajectory, `C` el número de contribuciones de Context de Trajectory materializadas y `Mᵣ` el total de Matches que sostienen los Contexts invalidados por un prepend. `D` es un conjunto registrado pequeño; los chunks de streaming se agregan en un Context de Assistant, de modo que `C` normalmente es mucho menor que `E`.

| Ruta | Trabajo de Context | Trabajo de instantánea objetivo | Resultado |
|---|---|---|---|
| Reemplazo inicial de tail o reconexión | Empareja la ventana cargada en `O(E × D)` y construye el State en orden de Event hacia delante | Construye y ordena `C` contribuciones | Un reemplazo completo sigue siendo proporcional a la ventana cargada |
| Prepend de página más antigua | Empareja solo los Events frescos y reproduce solo los Contexts cuyo Match, Location o respuesta de Reader cambió, en `O(P × D + Mᵣ)` | Reconstruye la instantánea de etapa desde `C` contribuciones | El pliegue de negocio no se reinicia sobre todos los `E` Events |
| Append en vivo | Empareja en `O(D)`, localiza el Context con clave en `O(1)` y actualiza solo ese State | Reemplaza una contribución de mismo ancla en `O(1)` antes del ensamblado de instantánea | La correlación de negocio es independiente del historial de Events cargado |

El Builder almacena las contribuciones por clave de Context y mantiene un índice clave-a-posición. Una actualización de contenido con el mismo ancla reemplaza una contribución en su lugar; una contribución nueva o un cambio de ancla reconstruye y ordena el orden de contribuciones. El ensamblado de instantánea recorre entonces `C` contribuciones, indexa los Request headers y los schemas de Tool con Maps, y maneja los límites de Compaction y los errores de Turn con cursores o índices lineales.

El ordenamiento final de Event y Request mantiene el límite superior actual de una publicación en `O(C log C)`. La migración elimina las búsquedas inversas repetidas y el re-pliegue antiguo de historia cruda, pero no reivindica publicación de extremo a extremo `O(1)`. Chat conserva su comportamiento y complejidad de instantánea con clave existentes; añadir el objetivo de Trajectory no hace que Chat escanee Contexts o Nodes de Trajectory.

### Rutas calientes de presentación independientes

La migración de Context y las siguientes optimizaciones de presentación resuelven costes distintos. Estas reducciones preservan el modelo de vista existente y son teóricas a partir de conteos de llamadas y comportamiento asintótico; esta decisión no reivindica mediciones de benchmark.

| Ruta caliente | Comportamiento retenido | Reducción esperada |
|---|---|---|
| Resúmenes de Markdown | El layout retiene el Markdown fuente; cada registro de Table estable memoriza su resumen mostrado por contenido, mientras que Detail analiza solo el registro seleccionado | Un append de un registro re-analiza el registro visible cambiado en lugar de cada registro de Markdown |
| Texto de búsqueda | `TrajectorySearchIndex` comprueba linealmente los IDs de registro estables y las firmas fuente, pero normaliza Markdown solo para los registros cambiados y confirma las actualizaciones en lotes de tres segundos | La comparación de firmas sigue siendo `O(C)`; la normalización costosa sigue el conteo de registros cambiados, y las actualizaciones continuas de frame colapsan en un lote por intervalo |
| Tooltip de Timeline | El texto de tiempo se calcula después de que el tooltip retardado se abre | Un render sin tooltip abierto no realiza formato de etiqueta por span |
| Búsqueda de Assistant siguiente | Un paso inverso registra el siguiente Assistant para cada posición de entrada | La antigua búsqueda directa repetida cae de `O(C²)` en el peor caso a `O(C)` |
| Duración de grupo | El agrupamiento decimal fijo reemplaza a `toLocaleString('en-US')` para la forma numérica inglesa invariante | La complejidad sigue siendo lineal en Groups, pero el formateador Intl sale de la ruta de render repetida |

La memorización de display y la indexación de búsqueda permanecen separadas. La búsqueda debe incluir registros fuera de pantalla y puede retrasarse respecto a los cambios en vivo por el intervalo de throttle; el render de Table debe actualizar inmediatamente el registro visible cambiado y no debe heredar la cadencia de confirmación del índice.

## Alternativas consideradas

**Mantener el pliegue independiente de Session History y optimizarlo localmente.** Rechazado: las cachés podrían reducir rutas calientes seleccionadas, pero Trajectory seguiría siendo dueño de una segunda ventana de Events, de la reparación de paginación, del pliegue de inspección de requests y de la implementación de correlación de negocio junto a Chat.

**Reutilizar las Definitions de Chat y bifurcar en un argumento `target` en `buildViewNode()`.** Rechazado: Trajectory necesita State y registros intermedios distintos, no solo otro renderer de React. Una Definition llevaría los payloads y condicionales de ambas vistas e invalidaría datos objetivo no relacionados cuando cualquiera de las dos vistas cambiara.

**Crear un Assembler específico de Trajectory.** Rechazado: el enrutado por ID exacto, la recolección de actualización-antes-de-inicio, la reproducción de prepend, la reparación de Location, las dependencias de Reader y la cadencia de publicación no son específicos de Trajectory. Un segundo motor recrearía la duplicación de ciclo de vida que este cambio elimina.

**Añadir conceptos genéricos de Surface, rewind, fanout o ciclo de vida asentado.** Rechazado: el stream de Events durables actual no exige una rama genérica de Surface, y los límites de Session o Turn son entradas de negocio del objetivo, no una razón para hacer fanout de un Event sobre todo Context histórico. La finalización sigue siendo State de negocio interpretado con cierre de Location.

**Reemplazar las etapas de Trajectory por Conversation Nodes genéricos.** Rechazado: las etapas organizan requests, tiempo, schemas y layout de tabla para una vista. Convertirlas en contratos de motor constreñiría una futura vista plana de Session-log y devolvería la composición específica de vista a Client Runtime.

**Compartir una caché de Markdown entre display y búsqueda.** Rechazado: el display es inmediato y acotado al viewport, mientras que la búsqueda cubre el conjunto completo de registros cargados y agrupa actualizaciones a propósito. Una caché compartida acoplaría corrección y programación entre consumidores no relacionados.

## Verificación

Las pruebas de runtime fijan el registro de objetivos, el append por ID exacto, la reproducción de actualización-antes-de-inicio, la identidad de prepend, la reparación de huecos de ventana del Reader, la reproducción de Location y el aislamiento entre instantáneas de Chat y Trajectory.

Las pruebas de Definition y Builder de Trajectory fijan el streaming e interrupción de Assistant, las llamadas de Tool anidadas y la interrupción paralela, la Compaction y la herencia de prompt, la clasificación de Steering y la colocación en Step, el orden de marcadores de Request, el reemplazo estable de contribuciones y la expansión de prepend. Las pruebas de Table, layout, Timeline y búsqueda fijan el trabajo de Markdown diferido, las actualizaciones de índice con throttle, el formato en tiempo de tooltip y los resultados de búsqueda estables entre append y prepend.

## Consecuencias

El ensamblado de negocio de Trajectory escala ahora con la página cambiada o el Context con clave en lugar de reiniciarse desde la ventana completa de Events crudos. Las Definitions propiedad del objetivo pueden evolucionar con independencia de Chat reteniendo una ventana de Session y un conjunto de reglas de ciclo de vida. El steering se convierte en un registro de Trajectory de primera clase en su posición real de Step sin añadir estado específico de steering a Session.

El Builder orientado a etapas retenido sigue realizando trabajo proporcional a las contribuciones de Trajectory materializadas y puede ordenar al publicar. El índice de búsqueda sigue realizando un paso de firmas lineal ligero cuando cambia su layout de entrada. Estos costes son trabajo explícito de vista objetivo, no re-pliegue oculto de Events completos.

Los autores de Definition deben proporcionar identidades de protocolo estables. Los Events antiguos sin el ID requerido pueden desaparecer de la vista de negocio de Trajectory afectada, lo que es preferible a unir registros no relacionados o a fallar la carga de historial; los productores que exigen display fiel deben registrar la identidad.

La [decisión de ensamblado de Conversation](2026-08-09-client-conversation-node-assembly.es.md) sigue siendo la autoridad para los contratos genéricos de Context, Reader, Location y publicación. La [decisión de registro de Trajectory](../feature/2026-07-27-trajectory-inspection-ledger.es.md) sigue siendo la autoridad para la jerarquía de tablas, la virtualización, el inspector y el comportamiento de interacción. Este Note es dueño de cómo Trajectory adapta esas dos decisiones y de por qué la adaptación no comparte Nodes finales con Chat.
