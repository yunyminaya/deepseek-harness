# @deepseek-ai/dsh-llm-replay

[English](README.md) | Español

Un plugin de LLM de replay para tests de instantánea sin clave. Produce streams de modelo reconstruidos a partir de un fixture de **session JSONL** grabado, de modo que un test puede arrancar el agent real contra un transcript de modelo fijo sin clave de API. Con `providers` configurado registra un adaptador solo de replay cuyo catálogo está disponible para los escenarios que ejercitan el descubrimiento de modelos; sin `providers` instala el waterfall `llm/stream` de captura total usado por los tests que no necesitan descubrimiento.

Sus consumidores son las suites de instantánea ACP y headless `stream-json`, más la línea e2e del navegador web. Las suites dirigidas por el Loader montan este plugin en lugar de un adaptador LLM real; la línea web lo instala directamente para conservar el handle de consumo del desmontaje.

## Cómo funciona el fixture

El fixture es una proyección de un log de sesión persistido (`<scenario>/session.jsonl`): conserva la cabecera y cada payload de evento, pero omite los envoltorios `seq`/`time` del cuerpo (`seq0`/`time0` para las filas empaquetadas). El replay restaura envoltorios sintéticos contiguos durante el parseo; un archivo debe contener filas de cuerpo proyectadas o filas de cuerpo persistidas completas, nunca ambas. La persistencia de runtime sigue escribiendo el log completo. Los eventos `assistant/chunk` del fixture llevan cada `StreamChunk`, así que agruparlos por `(turn, step)` reconstruye la secuencia de chunks de cada llamada `stream()` del agent loop. Un resumidor de compactación exitoso se registra de forma distinta: cuando `compaction/summary` lleva `llmStreamCall: true` y su `rawOutput` completo, el replay reconstruye un stream exitoso canónico en la posición de ese evento usando un par `block-start`/`block-end` por bloque, el uso registrado cuando está presente y un `stop` terminal. La partición exacta de deltas del provider no forma parte del resultado durable de compactación. `rawOutput` sin el marcador no implica una llamada LLM local porque los resumidores de plantilla y remotos pueden conservar la salida completa sin usar el adaptador de este contexto.

Grabar consiste por tanto en "ejecutar el agent real una vez y cosechar el `.jsonl`", tarea del harness de instantáneas — este plugin no graba. Un fixture puede llevar el contenido de su `request/header` tokenizado a `{{system}}`/`{{tools}}` (el harness fija ese contenido en un escenario y depura el resto); al replay le da igual — la derivación lee solo los eventos `assistant/chunk` y `compaction/summary` más la cabecera de sesión de la línea 0.

Dos modos de fallo no son reconstruibles solo desde `assistant/chunk` — un lanzamiento puro antes de cualquier chunk (p. ej. un HTTP 401, donde el log contiene solo un `turn/end {error}` y ningún chunk) y una cancelación/colgado (temporización, no contenido de chunks). Un escenario que los necesite suministra un sidecar opcional (`<scenario>/replay.override.json`) que o bien reemplaza el script derivado (un `ReplayEntry[]` puro) o bien lo aumenta (`{ patches: [{ at, entry }] }`: conserva cada llamada derivada del JSONL y permuta los índices de llamada nombrados basados en 0; un `at` igual a la longitud derivada añade el intento de reintento tras un lanzamiento transitorio inyectado). Los índices de parche deben ser únicos. El documento de override, cada parche y entry, y cada discriminante de chunk se validan cuando el archivo se carga. Una entry `hang` puede nombrar un `readyFile`; el replay escribe ese marcador vacío después de que sus chunks de prefijo lleguen al loop y antes de esperar la cancelación, de modo que un driver externo pueda cancelar de forma determinista sin observar una actualización de presentación.

Una cadena programada puede incrustar `{{fromRequest:<regex>}}` para rellenar un valor que ningún sidecar estático puede conocer — por ejemplo, un id de goal acuñado aleatoriamente que el modelo debe devolver a `update_goal`. En tiempo de stream cada placeholder se resuelve contra la petición viva: el corpus es cada hoja de cadena de los mensajes de la petición unidos por saltos de línea, gana la ÚLTIMA coincidencia del patrón en el corpus, y su primer grupo de captura (o la coincidencia completa sin uno) sustituye en su lugar. Un patrón que no coincide con nada, un patrón inválido y un placeholder sin terminar fallan todos de forma ruidosa. Las dos últimas llaves de una racha consecutiva de `}` terminan el placeholder, así que un patrón puede acabar con un cuantificador de llaves (`[0-9a-f]{4}`) pero no puede contener `}}` seguido de más contenido de patrón. La resolución se aplica a cada entry programada, incluidas las derivadas del JSONL grabado — un fixture grabado cuyo texto contiene legítimamente el marcador literal debe expresarse a través de un sidecar sin él.

## Agents anidados: claves por sesión

Un escenario donde un agent padre delega en subagentes en proceso graba más de un log: el padre (`session.jsonl`) más uno por hijo (`session.1.jsonl`, …). Cada agent se ejecuta como su propia `Session` en el mismo contexto, así que el replay debe servir a cada uno su propio script.

El replay asigna la clave de cada llamada por el id de sesión que la llama (`GenerateOptions.sessionId`, sellado por el agent loop). Los ids de sesión vivos son frescos y aleatorios en cada ejecución y nunca igualan los grabados, así que una sesión viva se vincula a un script grabado por **orden de primera llamada**: los scripts se ordenan por `createdAt` de la cabecera (el padre primero — transmite antes de poder delegar), y la primera sesión viva que hace cualquier llamada reclama el primer script, la siguiente sesión nueva el siguiente, y así sucesivamente. Cada sesión avanza entonces su propio cursor. Una llamada sin `sessionId` es una sesión anónima vinculada al script primario. Más sesiones vivas distintas que scripts grabados falla de forma ruidosa.

## Configuración

| Clave | Tipo | Por defecto | Notas |
|---|---|---|---|
| `file` | string | `$DSH_SNAPSHOT_FILE` | Ruta al fixture `session.jsonl` primario (padre). Obligatorio (config o env). |
| `overrideFile` | string | `$DSH_SNAPSHOT_OVERRIDE` | Sidecar `ReplayOverrideDoc` opcional para la sesión primaria: un `ReplayEntry[]` puro reemplaza su script derivado, mientras que `{ patches }` lo aumenta por índice de llamada. |
| `childFiles` | string[] | `$DSH_SNAPSHOT_CHILD_FILES` (delimitado por rutas) | Logs de sesión de subagentes hijos grabados para un escenario anidado; vacío para un escenario de una sola sesión. |
| `providers` | `ReplayProviderConfig[]` | — | Catálogo opcional de providers y modelos solo de replay. Cada provider puede fijar `retryPolicy`, y cada modelo puede publicar `contextWindow` y un array `inputModalities` que contenga solo `text` e `image`; las modalidades inválidas fallan durante la carga del plugin. Las rutas configuradas se despachan a través del adaptador de replay y nunca realizan I/O de provider. |
| `paceMs` | number | — (burst) | Retraso opcional por chunk en ms para que los transports posteriores (p. ej. el mux SSE web observado por un navegador real) vean una entrega genuinamente incremental. Solo un mando de realismo — los tests no deben depender de él para la corrección. Entero no negativo; un abort durante una espera de pace cancela el stream con prontitud. |

```yaml
- id: llm-replay
  name: '@deepseek-ai/dsh-llm-replay'
  config:
    providers:
      - id: deepseek-official
        name: DeepSeek
        retryPolicy:
          mode: normal
          backoff:
            initialDelayMs: 1
            maxDelayMs: 1
            jitterRatio: 0
        models:
          - id: deepseek-v4-flash
            contextWindow: 128000
          - id: deepseek-v4-pro
  # file/overrideFile/childFiles default to $DSH_SNAPSHOT_FILE /
  # $DSH_SNAPSHOT_OVERRIDE / $DSH_SNAPSHOT_CHILD_FILES, set by the snapshot
  # harness per scenario.
```

## Exportaciones

- `installLlmReplay(ctx, config)` — instala el adaptador de replay configurado o el listener `llm/stream` de captura total; devuelve un `ReplayHandle` (`dispose()` para seguridad HMR más `assertConsumed()`, la comprobación de desmontaje de que cada script grabado se vinculó a una sesión viva y cada cursor vinculado se drenó — convirtiendo un escenario que condujo silenciosamente menos llamadas de modelo que las grabadas en un diagnóstico nítido). Úsalo en tests para conducir el replay sin el Loader ni variables de entorno.
- `loadSessionScripts(config)` — resuelve los `SessionScript[]` ordenados (primario + hijos) para un escenario, listos para vincularse a sesiones vivas en orden de primera llamada.
- `loadReplayScript(config)` — resuelve los `ReplayEntry[]` solo para la sesión primaria (sidecar de reemplazo/parches validado si está presente; si no, derivado del JSONL; falla de forma ruidosa si falta el fixture).
- `deriveReplayScript(events)` / `parseSessionLog(text)` / `parseSessionHeader(text)` / `resolveScriptedEntry(entry, messages)` — los helpers puros que convierten chunks ordinarios del loop y salidas de compactación local marcadas explícitamente en un log de sesión persistido o proyectado en un script, leen su `id`/`createdAt` de cabecera y resuelven los placeholders `{{fromRequest:...}}` contra una petición viva. Un grupo de assistant derivado debe terminar en un chunk `finish`; un grupo sin uno es la huella de un `stream()` lanzado y debe expresarse en su lugar mediante un sidecar de override.
- Tipos `ReplayEntry` / `ReplayOverrideDoc` / `ReplayOverridePatch` / `SessionScript` / `ReplayConfig` / `ReplayProviderConfig` / `ReplayModelConfig` / `ReplayHandle` / `Config`.

## Forma de exportación del plugin

`name` / `inject` / `Config` / `apply` con nombre, **sin export por defecto**: el `unwrapExports` del Loader de Cordis hace `exports.default ?? exports`, así que un default accidental colapsaría el módulo a la función desnuda y perdería el namespace `inject` (ver [docs/postmortem/0001](../../../docs/postmortem/0001-acp-default-export-drops-inject.es.md)).

## Model Experience

Ninguna, ya que este adaptador de prueba sin clave no envía petición a un modelo de provider; solo reproduce chunks de assistant grabados en el loop de prueba.

#### Efecto de caché KV

Ninguno; este paquete ni ensambla ni envía una petición de provider.

## Limitaciones conocidas y trabajo diferido

- **La vinculación de scripts por orden de primera llamada asume delegación secuencial** — una versión que ejecutara subagentes hermanos en concurrencia vincularía las sesiones vivas a los scripts grabados de forma no determinista; una clave más fuerte queda diferida hasta que exista tal escenario (`XXX(concurrent-subagents)`).
- **Solo los chunks ordinarios del loop y las salidas de compactación local marcadas son derivables** — un lanzamiento puro anterior al chunk, una cancelación/colgado o una llamada de resumidor externo sin marcar necesita el sidecar `replay.override.json`. Las formas de reemplazo y parche afectan solo a la sesión primaria; los scripts hijos siguen derivándose de sus logs.
