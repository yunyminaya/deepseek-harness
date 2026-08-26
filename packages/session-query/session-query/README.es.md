# @deepseek-ai/dsh-session-query

[English](README.md) | Español

`SessionQueryEngine` es el contrato abstracto combinado `ctx.sessionQuery`. Implementa la recuperación exacta del historial de sesión, el rastreo de relaciones y el filtrado independiente del provider sobre las `ctx.sessions` en vivo más la `ctx.sessionPersistence` opcional montada dinámicamente; los backends concretos implementan sus dos métodos de texto completo. Los ids coincidentes producen un único registro: ganan los eventos en vivo, mientras que `live` y `persisted` informan de ambas disponibilidades de fuente. Las cabeceras inmutables en conflicto fallan con `SESSION_QUERY_SOURCE_CONFLICT`.

## Lecturas

- `listSessions(signal?)` lee los metadatos de persistencia actuales, fusiona los registros en vivo con precedencia de los vivos y devuelve registros clonados en orden determinista de más nuevo a más antiguo.
- `readSession(sessionId)` devuelve un log crudo completo y separado tras la misma validación de reproducción del núcleo que usa el resume; nunca introduce la sesión en el store en vivo.
- `filterSessions(filters, signal?)` aplica predicados de metadatos de sesión y de disponibilidad independientes del provider a ese mismo corpus lógico clonado.
- `filterEvents(sessionId, filters)` extrae los documentos semánticos propios y aplica predicados de metadatos y de texto literal independientes del provider en orden ascendente de seq.
- `readTitleSnapshots(sessionIds, signal?)` resuelve ids únicos a partir de una única observación del corpus con preferencia por lo vivo, propaga la cancelación a través del listado y la inspección persistidos, y devuelve asentamientos ordenados por sesión de modo que una fuente de títulos ausente o malformada no descarte a sus compañeras. Cada fuente viva se pliega directamente, y cada worker persistido se pliega hasta un resultado de cabecera/título separado y libera el log completo antes de sacar otro id de la cola. La cancelación rechaza todo el lote. `readTitleSnapshot(sessionId, signal?)` es la vista de una sola observación; `readTitle(sessionId, signal?)` devuelve solo su `session/title` plegado opcional.
- `listEvents(sessionId)` carga el log crudo con preferencia por lo vivo y clasifica cada evento como `current`, `shadowed` o `log-only` con el plegado de surface compartido de `dsh-session`.
- `readSurface(sessionId)` devuelve una cabecera clonada, la frontera de captura del log crudo y la surface actual completa plegada en orden de historial del modelo. Una sesión viva gana sobre la persistencia; la compactación se observa antes o después de su anexión de reemplazo, nunca como mezcla sintética.
- `readEvent(request, signal?)` devuelve una cabecera clonada, el evento objetivo completo y una ventana acotada de seq crudo. `before` y `after` tienen por defecto cero y no pueden superar `readWindowMax`.
- `traceSession(sessionId, signal?)` lee el corpus una sola vez y devuelve los ancestros inmediatos hacia fuera más los árboles de descendientes recursivos deterministas. `complete: false` identifica el primer padre ausente; un ciclo conectado al objetivo falla con `SESSION_QUERY_INVALID_LINEAGE`.
- `traceEvent(request, signal?)` carga el log lógico una sola vez y devuelve su cabecera de fuente clonada con reemplazos posicionales directos y enlaces directos a los eventos de fuente citados. `replacementChain` sigue a los reemplazadores posicionales hasta el reemplazo final; los enlaces a eventos de fuente no son transitivos.

La persistencia es opcional y puede montarse o desmontarse dinámicamente. El listado entre corpus y el rastreo de linaje fallan con `SESSION_QUERY_PERSISTENCE_FAILED` mientras la persistencia montada es ilegible; un registro duradero leído correctamente que no supere la validación de Session informa en su lugar `SESSION_QUERY_CORRUPT_SESSION`. Una lectura de título, un rastreo de evento o una lectura de evento dirigidos a una sesión viva conocida no consultan la persistencia, de modo que un backend duradero enfermo no puede volver ilegible el estado en memoria actual. Las operaciones persistidas de títulos y eventos listan antes de cargar y rechazan una discrepancia de metadatos en lugar de combinar observaciones incoherentes. La cancelación del rastreo de linaje se propaga al listado persistido; la cancelación del rastreo de eventos y de la lectura de eventos se propaga al listado y a la inspección persistidos. Cada una espera a que se asiente la llamada al backend ya iniciada y después rechaza con la razón exacta de la señal aunque el backend la haya ignorado. Una lectura de título, un rastreo de evento o una lectura de evento sobre una sesión viva conocida pre-abortada rechaza antes de plegar o de tomar instantáneas sin consultar la persistencia. Una observación de títulos por lotes realiza un único listado de metadatos, inspecciona sus ids persistidos únicos con como máximo `persistedInspectConcurrency` workers y conserva la cabecera observada propia de cada título para la autorización posterior. La cancelación no inicia inspecciones encoladas y solo rechaza después de que se asienten los workers ya iniciados. `listSessions()` sigue siendo ligera: no carga logs ni indexa títulos.

## Filtrado y extracción

`SessionResultFilter` cubre el id, el cwd anulable, el rango de created-at, el padre anulable y la disponibilidad de fuente. `SessionEventResultFilter` cubre los rangos de seq/tiempo, el tipo de evento, la surface y el texto semántico. Los arrays de filtros se combinan con AND; los valores dentro de una cláusula de lista se combinan con OR. Los valores de lista vacíos no coinciden con nada, los rangos son inclusivos y los rangos malformados o los valores de unión cerrada fallan con `SESSION_QUERY_INVALID_FILTER`.

La cláusula de texto es deliberadamente independiente de los providers de FTS: el texto del llamante se escapa a una expresión regular Unicode insensible a mayúsculas, y cada secuencia de espacios en blanco coincide con uno o más caracteres de espacio en blanco. Es un escaneo literal de texto semántico, no una consulta de texto completo. `extractSessionEventText()` y `buildSessionEventSearchDocuments()` definen la proyección de documento semántico propia compartida; los bloques de razonamiento, las fronteras estructurales, los fragmentos de stream, las cabeceras de solicitud y las variantes desconocidas de declaración fusionada no producen documento.

## Métodos de texto completo

`SessionQueryEngine.searchSessions(request, exec?)` agrupa el corpus lógico por el evento que mejor coincide; `searchEvents(request, exec?)` busca en una única sesión lógica. Son los únicos métodos abstractos del servicio. Ambos devuelven páginas cuya continuación es un `SessionSearchCursor` marcado propiedad del servicio, aceptan cancelación opcional y exponen fragmentos sin puntuaciones numéricas específicas del provider. Una página de búsqueda de eventos lleva también la cabecera objetivo clonada de la misma generación indexada que sus coincidencias, lo que permite a los Consumers de autorización vincular la política a la observación del payload. Las solicitudes de búsqueda solo aceptan filtros de eventos de metadatos, porque el filtrado por texto literal es la vía de escaneo descrita arriba.

El paquete no tiene coordinador de providers, implementación de fallback ni plugin concreto independiente. Un backend de servicio concreto hereda las lecturas, los filtros y los rastreos implementados mientras es dueño de la observación de texto completo, la reconciliación, el ranking, las generaciones de cursor y la ejecución de consultas; la primera implementación es [`@deepseek-ai/dsh-session-query-sqlite`](../session-query-sqlite/README.es.md).

`SessionQueryError.code` es una unión cerrada que cubre la validación de solicitudes, los objetivos ausentes, las surfaces malformadas, los conflictos de fuente, los fallos de persistencia/índice, la cancelación y los cursores inválidos u obsoletos; los literales exactos están definidos en [`src/config.ts`](src/config.ts).

`listEvents()`, `readSurface()` y `traceEvent()` ejecutan el mismo plegado de surface de una sola pasada de `dsh-session`. Un log cargado solo es válido cuando los seq de los eventos están basados en cero y son contiguos, los marcadores de surface respetan la elegibilidad por tipo de evento, los arrays de eventos de fuente no están vacíos ni tienen duplicados, las referencias nombran eventos anteriores y cada reemplazo posicional nombra y cita cada nodo de surface que elimina; cada violación falla con `SESSION_QUERY_INVALID_SURFACE`.

## Configuración

| Clave | Valor por defecto | Contrato |
|---|---:|---|
| `readWindowMax` | `50` | Número máximo de eventos crudos `before` o `after`. |
| `persistedInspectConcurrency` | `4` | Máximo de inspecciones concurrentes de logs persistidos en una lectura por lotes; debe ser un entero seguro positivo. |

## Experiencia de modelo

Ninguna: este servicio de consulta de confianza devuelve registros de sesión clonados solo a sus llamantes y no registra prompt, schema, herramienta ni mensaje orientados al modelo.

#### Efecto en la caché KV

Ninguno; este paquete ni ensambla ni envía una solicitud de provider.

## Limitaciones conocidas y trabajo diferido

- **Sin autorización del llamante** — es infraestructura de confianza de todo el contexto; una futura herramienta o interfaz de modelo debe limitar qué sesiones puede inspeccionar su llamante.
- **Sin registros ni herramienta orientada al modelo** — faltan los registros de extractores y de search providers, el recorrido recursivo a través de los eventos de fuente citados y una herramienta orientada al modelo. La [decisión de rastreo](../../../.agents/notes/implemented/feature/2026-07-13-session-query-tracing.md) es dueña de la semántica de relaciones; las decisiones de propiedad de SQLite y de tokenizer viven en la [nota de búsqueda implementada](../../../.agents/notes/implemented/feature/2026-07-10-sqlite-session-query-provider.md).
