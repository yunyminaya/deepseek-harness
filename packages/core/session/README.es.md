# dsh-session

[English](README.md) | Español

Log de sesión event-sourced y store en memoria. Una `Session` es la fuente de verdad de solo añadido para todo el historial de interacción de un agent — el historial de mensajes del LLM (modelo de lenguaje de gran tamaño) se *deriva* de ella. Sobre el log crudo se mantiene una capa **surface** (una proyección ordenada de los eventos que producen mensajes) para la derivación y la compactación eficientes.

El complemento opcional `@deepseek-ai/dsh-session/invariant` registra las comprobaciones de traza relacional de este paquete con `ctx.invariants`: números de secuencia monótonos, contención turno/step y emparejamiento llamada de herramienta/resultado del mismo step. Reproduce las sesiones existentes al cargarlas o recargarlas; la validación del almacenamiento, la toma de instantáneas, la congelación, la validación de los eventos fuente citados y la aceptación de la surface siguen siendo responsabilidades siempre activas del paquete raíz de sesión.

## Service: `SessionStore` (ctx key: `sessions`)

Crea y mantiene instancias `Session` event-sourced. La persistencia no se implementa aquí a propósito — los plugins se suscriben a `session/event`, hacen flush en `session/flush` y pueden reflejar el ciclo de vida emparejado `session/created`/`session/disposed`.

### Public API

- `ctx.sessions.create(id?, { seed?, meta? }?)` valida y desacopla los datos duraderos de seed/header, rellena la versión y el id, toma `createdAt` como ahora por defecto, publica la sesión y la vincula al fiber llamador. La reconstrucción persistida suministra su `createdAt`, `seedLength` y `delegationDepth` originales.
- `ctx.sessions.flush(session)` despacha el punto de control de durabilidad paralelo esperado a través del scope capturado de la sesión. Todos los listeners arrancan y la llamada espera a que todos se resuelvan antes de informar del fallo; los objetos sin publicar, desacoplados y obsoletos se rechazan.
- `ctx.sessions.fork(source, boundary?, childSessionId?): Session` — Resuelve un objeto o id de sesión vivo, selecciona un seed hasta el seq de evento `boundary` inclusivo (por defecto: el último evento actual), exige que ese prefijo termine fuera de un turno abierto y crea una sesión hija viva con metadatos de linaje.
- `ctx.sessions.get(id: SessionId): Session | undefined`
- `ctx.sessions.list(): Session[]`

#### Avanzado: primitivas de ciclo de vida con teardown ordenado

Usa el ciclo de vida separado solo cuando el teardown deba ordenarse con otro recurso:

- `prepare(id?, options?)` valida y construye sin publicar.
- `enter(session)` realiza la comprobación de colisión, publica sin anunciar y devuelve un detach idempotente vinculado a la entrada. Se permiten preparaciones concurrentes con el mismo id, pero solo una entrada tiene éxito; un detach obsoleto no puede eliminar a su reemplazo.
- `announce(session)` emite el único borde de creación y rechaza los anuncios repetidos o reentrantes. El detach durante ese dispatch se difiere y emite después el borde de eliminación emparejado; una entrada no anunciada no emite ninguno de los dos bordes de ciclo de vida.

`dsh-agent-loop` usa esta separación para que el flush final del loop preceda al detach de la sesión; consulta la [Agent Note de propiedad](../../../.agents/notes/implemented/architecture/2026-06-18-agent-lifecycle-and-ownership-contracts.md).

### Eventos de servicio en vivo

El store empareja la creación anunciada con la eliminación, publica notificaciones de añadido post-commit con contención por listener y proporciona un punto de control de durabilidad esperado. Las firmas exactas y el comportamiento de scope viven en la región generada de [session.md](../../../docs/subsystems/session.es.md#cordis-surface); los payloads viven en el [catálogo de persistencia](../../../docs/persistence-catalog.es.md).

### Clase: `Session`

Clase simple (no un Service de Cordis). Crea las sesiones vivas mediante `ctx.sessions.create()` y las sesiones desacopladas de reproducción o inspección mediante `Session.create()`; la fábrica desacoplada no publica eventos de ciclo de vida ni vincula la sesión a un fiber.

- `session.append(type, data, opts?)` toma una instantánea y congela los datos duraderos y los metadatos de surface, valida la forma de los marcadores, los seqs de eventos fuente citados, la cobertura completa del reemplazo y las reescrituras `tool/result` de resultado único solo de contenido, confirma de forma síncrona y notifica después a los observadores con contención de fallos independiente. Los añadidos reentrantes a sesiones adjuntas se rechazan, y las comprobaciones de runtime cubren las uniones ampliadas y los logs cargados.
- `session.deriveMessages()` proyecta de forma incremental cada entrada de surface nueva una sola vez y devuelve un array nuevo sobre los mensajes completos, identificados y congelados que almacenan esas entradas. Los mensajes de asistente preservan en su source de modelo el provider y el modelo que los produjeron además del estado de reproducción privado del adaptador. Una reescritura de surface reconstruye la proyección; no hay respaldo de log crudo.
- `session.deriveEventMessage(event)` es la proyección canónica por evento usada por la reconstrucción y las comprobaciones de solicitud.
- `session.surface` expone la vista de solo lectura `SessionSurface` propiedad del único gestor de surface incremental de la sesión; `replaceGeneration` cambia en cada reescritura confirmada.
- `session.events` es una instantánea congelada en caché invalidada por el añadido; los eventos aceptados permanecen profundamente congelados.
- `session.seq`, `session.id` — la secuencia actual y la identidad tipada de solo lectura.
- `session.header: SessionHeader` — metadatos de creación desacoplados y profundamente congelados (`version`, `id`, `createdAt`, `cwd`/`parentSession`/`seedLength`/`delegationDepth` opcionales). La construcción valida el registro duradero y exige que su id coincida con `session.id`.

### Utilidades JSON sin pérdida

Los valores duraderos necesitan una única representación aceptada, no una comprobación seguida de una segunda lectura. `isJsonValue(value)` es el predicado booleano; `snapshotJsonValue(value)` valida y copia de forma iterativa un valor simple en una sola pasada, devolviendo `undefined` para la entrada no válida y propagando un getter que lanza. El helper de instantánea acepta números JSON finitos excepto `-0` (JSON lo reescribe como `0`), arrays ordinarios densos y objetos simples o con prototipo null; rechaza los ciclos, los escalares no admitidos y los prototipos exóticos antes de la normalización sin imponer un límite de profundidad de pila de llamadas.

La importación de eventos de sesión separa la propiedad de la validación de mensajes. `snapshotSessionEvent(event)` clona un evento prestado antes de validar y congelar su mensaje identificado. `adoptSessionEvent(event)` realiza el mismo trabajo de mensaje in situ y devuelve el evento original; los llamadores pueden usarlo solo cuando transfieren un grafo de objetos de propiedad exclusiva sin ningún hijo mutable compartido con otro evento.

### Codec de almacenamiento por filas de chunk (`chunk-rows.ts`)

El [codec de almacenamiento](src/chunk-rows.ts) compartido convierte sin pérdida las secuencias de eventos en filas compactas y viceversa. Preserva los eventos no reconocidos tal cual y rechaza las filas codificadas malformadas; los backends de persistencia deciden si habilitan las escrituras empaquetadas.

### Tipos de surface

Este paquete es dueño de la proyección ordenada de surface, la validación de reemplazos, la reproducción y las guards de tipo que distinguen los eventos de origen de añadido de los de reemplazo. El [catálogo de tipos de surface](../../../docs/subsystems/session.es.md#surface-types) es el dueño de las formas exactas y de la semántica de los campos. Un transcript (transcripción) humano debe proyectar los eventos de origen de añadido en lugar de `session.surface`, porque los reemplazos aterrizados ensombrecen el historial que el lector ya vio; los consumidores orientados al modelo siguen leyendo `session.surface`.

### Reconstrucción de la cabecera de solicitud (`request-header.ts`)

`request/header` registra una instantánea canónica completa del envoltorio de solicitud que no es historial con razón `initial`, `resume` o `change`. Su mapa opcional `adapterDefaults` marca los valores efectivos de `reasoningEffort` o `maxTokens` materializados por la resolución de modelo exacto, lo que permite a la siguiente propuesta de solicitud distinguirlos de los ajustes explícitos de la conversación. `foldRequestHeader()` selecciona la instantánea más reciente; los eventos delta heredados y la razón `fallback` eliminada se rechazan. Consulta la [Agent Note de solicitudes reconstruibles](../../../.agents/notes/implemented/architecture/2026-07-05-reconstructable-requests.md).

Un `user/message` almacena el `UserMessage` completo directamente, incluida la identidad creada antes del enrutado al inbox o de la entrada al step. Renderiza su `content` tal cual, ya sea un prompt humano directo, una inyección sintética o una ronda de goal entrado; su `source` tipado es el único canal que los distingue y transporta cualquier hecho duradero específico del dominio. `assistant/message` y `tool/result` almacenan igualmente valores de mensaje completos. La ejecución del turno permanece encerrada por `turn/start` y `turn/end`; `agent.inject()` pone en cola la entrada hasta que un pre-step posterior la reclama y la devuelve en una decisión enter.

`tool/result` persiste un mensaje de resultado de herramienta identificado con rol de usuario, una identidad de fallo interno opcional y metadatos de presentación opcionales. El `value` canónico de éxito de una herramienta y su mensaje de fallo canónico legible por humanos permanecen locales a la ejecución; el contenido de error renderizado es el mensaje autoritativo para la reproducción.

### Vocabulario de eventos de sesión (`types.ts`)

El [catálogo de eventos del log de persistencia](../../../docs/persistence-catalog.es.md) generado enumera cada tipo de evento de solo añadido con su payload, su distintivo de surface y su lugar de declaración. La contabilidad de tokens lee los registros `assistant/chunk { type: 'usage' }` por step y trata `assistant/message.usage` como respaldo del step confirmado cuando no existe ningún chunk de usage; los intentos fallidos de solicitud de modelo no tienen mensaje de asistente. Cada `assistant/message` registra el provider, el modelo y el estado de reproducción opcional.

Extensible por fusión mediante `SessionEventMap` — un plugin fusiona por declaración sus propios tipos (los `compaction/*` del seam de compactación, el `llm/retry` no surface de la recuperación acotada, los `hook/*` de los bridges de hooks); los miembros fusionados aparecen en el mismo catálogo. Un plugin es dueño de la invariante relacional de sus eventos fusionados, incluido si un evento solo de log puede aparecer entre turnos. Un productor que requiera durabilidad añade a través de `Session` y espera después `ctx.sessions.flush(session)` sin fabricar un turno de ejecución.

También define `TurnEndReasonMap`, el tipo suma etiquetado por `kind` extensible por fusión para los finales de turno. `turn/start` transporta solo el número de turno; el lote `user/message` entrado posterior registra su entrada, mientras que `llm/retry` registra la recuperación de la solicitud.

Un turno vivo interrumpido termina con `{ kind: 'aborted', reason: AgentCancelCause }`, preservando la causa de cancelación tipada en el transcript (transcripción) duradero. La persistencia importa el resultado aborted grueso del formato anterior admitido como `{ kind: 'aborted', reason: { kind: 'legacy' } }`, porque ese registro no retuvo a su llamador. Un fallo de turno transporta `{ kind: 'error', error }`; solo la recuperación tras un crash sintetiza `{ kind: 'interrupted' }`.

Cada `SessionEvent` transporta tres campos opcionales de nivel superior (metadatos estructurales):

- `sourceEventSeqs?: number[]` — números de seq de eventos anteriores citados como fuentes (p. ej., los seqs de `assistant/chunk` detrás de un `assistant/message`, o las entradas ensombrecidas detrás de una entrada de reemplazo de compactación). En `assistant/message`, un `[]` presente registra un stream de provider conocido como vacío, mientras que la omisión significa que un evento heredado o ajeno no registró el stream de la fuente; los demás eventos de surface exigen una lista no vacía cuando este campo está presente.
- `surfaceOp?: SurfaceOp` — cómo entró este evento en la surface. Ausente para los eventos no surface (límites, chunks, usage, errores).
- `ignorable?: true` — marca un evento que un lector puede omitir con seguridad cuando no reconoce el tipo; ausente significa obligatorio, de modo que un evento de tipo desconocido rechaza la reconstrucción de la sesión ([mecanismo](../../../.agents/notes/implemented/architecture/2026-08-10-session-log-version-mechanism.md)).

### Tipos de metadatos (`types.ts`)

- `SessionHeader` — metadatos de sesión escritos una sola vez al publicarse como `Session.header`, donde el desacople y la congelación profunda imponen la inmutabilidad en tiempo de ejecución: `{ version, id, createdAt, cwd?, parentSession?, seedLength?, delegationDepth? }`. Los loaders de persistencia pueden devolver copias desacopladas mutables del mismo tipo de datos. Vive aquí (junto a `SessionId`) porque `Session.header` está tipado por él; los backends de persistencia lo reexportan en lugar de poseerlo (lo que forzaría un ciclo de paquetes).

### Puntos de extensión

- Plugins de persistencia: suscríbete a `session/event` (write-behind) y drena en `session/flush` (esperado) y en la eliminación del fiber. Un backend duradero lee el log y lo recarga en una sesión viva; el contrato de metadatos (`SessionHeader`, `session.header`) es lo que un backend así almacena junto al log.
- Reproducción/fork: `create(id, { seed })` valida y congela un log contiguo del formato actual y reconstruye su surface; las cabeceras de solicitud exigen provider/model, y los mensajes de asistente exigen procedencia de provider/model. La persistencia es dueña de la compatibilidad de lectura antes de construir este seed del formato actual. `fork(source, boundary?, childSessionId?)` selecciona un prefijo de turno completado y registra el linaje.
- Compactación: `dsh-compaction-basic` añade un reemplazo `user/message` para los puntos de control de resumen, mientras que `dsh-compaction-tool-result-pruner` añade un reemplazo `tool/result` solo de contenido. La política de límites del emparejamiento de herramientas y su caché pertenecen al [seam `dsh-compaction`](../../compaction/compaction/README.es.md), mientras que este paquete es dueño de la pertenencia ordenada a la surface, la validación de reemplazos y `replaceGeneration`.

## Experiencia de modelo

### Historial de mensajes derivado

#### Lo que ve el modelo

El modelo recibe tal cual los mensajes completos de las entradas de surface `user/message`, `assistant/message` y `tool/result`. Sus identidades, roles, sources y bloques de contenido son los mismos valores establecidos en la creación; las proyecciones no acuñan identidades. Los prompts directos y el contexto inyectado siguen siendo eventos `user/message` separados cuyos sources preservan su procedencia. Un envoltorio de prompt cambia solo la presentación humana; su contexto de prefijo y su delimitador de solicitud ya están presentes en el contenido del evento. Las llamadas de herramienta viven dentro de los mensajes de asistente. Los chunks, los límites, el usage, los registros de hooks, los registros de todo y demás eventos solo de log no añaden ningún mensaje.

#### Efecto en tokens

Las entradas de surface añadidas se reenvían en los steps posteriores. Una operación de surface `replace` elimina las entradas ensombrecidas de las entradas futuras sin borrar sus registros crudos del log.

#### Efecto en la caché KV

Las entradas de surface añadidas preservan los prefijos reutilizables. Una operación `replace` invalida la reutilización desde el primer mensaje ensombrecido aunque el log de eventos subyacente siga siendo de solo añadido.

### Resultado de reparación tras crash

#### Lo que ve el modelo

Si la recuperación encuentra una solicitud de herramienta de asistente sin un `tool/call` duradero, su resultado sintético `TOOL_NOT_STARTED` dice `The tool call was interrupted before the Harness recorded it as started. Retry it if it is still needed.` Si un `tool/call` duradero no tiene resultado, su resultado `TOOL_OUTCOME_UNKNOWN` dice `The tool call was interrupted after it was recorded, but no result was durably recorded. Its outcome is unknown. Decide whether to retry from the tool semantics: retry only if the operation is read-only or idempotent; if it may have side effects, first verify external state or ask the user. Do not retry blindly.`

#### Efecto en tokens

Cero tokens en una sesión intacta. Cada llamada reparada añade su texto de error retenido específico del riesgo al reanudar.

#### Efecto en la caché KV

Solo añadido; el contenido recién visible sigue el prefijo de solicitud reutilizable y no invalida las entradas existentes de la caché KV.

### Cabecera de solicitud registrada

#### Lo que ve el modelo

La sesión reconstruye el prompt del sistema, los schemas de herramientas, la config de llamada y el prefijo de sesión que el loop envió realmente. Los eventos de cabecera no añaden una segunda copia al historial de mensajes; el prefijo se antepone fuera de `deriveMessages()`.

#### Efecto en tokens

Cero tokens duplicados por el registro. El prefijo reconstruido, el texto del sistema y los schemas siguen incurriendo en su coste normal por solicitud.

#### Efecto en la caché KV

El registro no provoca invalidación, y la reconstrucción exacta preserva la identidad del prefijo de solicitud. Una cabecera posterior con el prefijo, el prompt o los schemas cambiados puede invalidar la reutilización desde su primera diferencia.

## Limitaciones conocidas y trabajo diferido

- **Ramificación/árbol de sesiones** (árbol de entradas estilo pi) — diferido salvo que se necesite más allá del `fork()` basado en límites.
- **`fork()` corta solo en los límites estables de las sesiones vivas** — el prefijo seleccionado debe terminar fuera de un turno abierto y la fuente debe estar en el store; bifurcar una sesión persistida pero no cargada queda excluido de la [API de fork](../../../.agents/notes/implemented/feature/2026-06-30-session-store-fork-api.md).
- **`SESSION_FORMAT_VERSION` permanece fijada en `0`** — prerelease, sin compatibilidad amplia implicada: `Session` acepta solo las formas de seed actuales, y un backend rechaza cualquier otra versión nombrando la dirección (más nueva: "written by a newer harness — upgrade"; más antigua: "no upgrade path ships yet"). Los tipos de evento desconocidos se rechazan del mismo modo salvo que se marquen `ignorable` en el envoltorio; el mecanismo de versionado es la [nota de session-log-version-mechanism](../../../.agents/notes/implemented/architecture/2026-08-10-session-log-version-mechanism.md). Las actualizaciones estrechas de importación de almacenamiento pertenecen al límite de persistencia ([política](../../../AGENTS.md), [recuperación de mensajes pre-identidad](../../../.agents/notes/implemented/bug-fix/2026-07-28-load-pre-identity-session-messages.md)).
- **`TurnEndReasonMap` omite las variantes `refusal` / `max_turn_requests` nombradas por ACP** — limitadas por el productor: llegan cuando un adaptador o el loop las emite por primera vez.
