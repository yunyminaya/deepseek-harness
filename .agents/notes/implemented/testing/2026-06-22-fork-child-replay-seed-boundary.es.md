# Agent Note: Persiste el límite de la seed para que el replay del hijo de fork se enrute correctamente

Status: implemented

[English](2026-06-22-fork-child-replay-seed-boundary.md) | Español

## Problema

El [Agent Note de replay de instantáneas por sesión](2026-06-22-subagent-snapshot-replay.es.md) hizo que el nivel de instantáneas expresara una forma de agente anidado: un padre más un registro grabado por cada subagente en proceso, cada uno reproducido como su propio script con clave por sesión llamante. Señalaba (§ Alcance, última viñeta) que una instantánea de fork era «una adición trivial futura, no una brecha en el claveado». Eso estaba equivocado respecto a un hijo de fork específicamente — no el claveado, sino la *derivación del script*.

Un script de subagente se deriva de un registro de sesión grabado mediante [`deriveReplayScript`](../../../../packages/test-support/llm-replay): agrupa los eventos `assistant/chunk` del registro por `(turn, step)` en una entrada de replay por llamada a `stream()`. Esto es correcto para un hijo de **spawn**, cuyo registro contiene solo sus propias llamadas de modelo.

Un hijo de **fork** es distinto. El backend de fork siembra la sesión hija con un *prefijo balanceado de turnos completados del registro del padre* ([`dsh-subagent-in-process-driver`](../../../../packages/subagent/subagent-in-process-driver)), y esa seed se convierte en el `log` persistido de la sesión hija (el constructor de `Session` copia la seed en `this.log`). Así que el `.jsonl` de un hijo de fork comienza con los eventos del **padre** — incluidos los eventos `assistant/chunk` del padre — y solo después lleva el turno propio del hijo.

Derivar el script del hijo de todo el registro del hijo de fork reproduce por tanto las respuestas **grabadas del padre** como llamadas de modelo del **hijo**: la primera `stream()` del hijo de fork en vivo recibiría la primera secuencia de chunks grabada del padre en lugar de la suya. Los escenarios grabados son hoy todos de spawn, así que esto nunca disparó — pero una instantánea de fork se habría mal-enrutado silenciosamente, exactamente la clase de bug que el nivel de instantáneas existe para atrapar.

## Decisión

Registra dónde termina el prefijo **heredado** de una sesión, persístelo, y haz que el harness de replay derive el script de un hijo solo de sus **propios** eventos.

### 1. `seedLength` en la cabecera de la sesión

`SessionHeader` gana un `seedLength: number` opcional — cuántos eventos iniciales se heredaron vía una seed en lugar de producirlos esta sesión. El backend de fork lo estampa (= la longitud del prefijo sembrado) cuando crea el hijo; un spawn fresco lo deja ausente (≡ 0). Se enhebra a través de `CreateSessionOptions.meta` (y `CreateAgentOptions.meta`), fijado en `SessionStore.prepare`.

`seedLength` es **explícito**, nunca inferido de `seed.length`. Una reconstrucción (resume/load) siembra la sesión con TODO su registro almacenado, así que `seed.length` allí es la longitud completa, no el límite original — el camino de resume pasa de vuelta el `seedLength` persistido desde la cabecera cargada. (Misma forma que `createdAt`, que también se preserva explícitamente en la reconstrucción en lugar de re-asignarse a ahora.)

### 2. Ambos backends de persistencia lo hacen ida y vuelta

- **JSONL**: un campo `seedLength` en la línea de cabecera (`toHeaderLine`/`fromHeaderLine`).
- **SQLite**: una columna `seed_length` en la tabla `sessions`.

El layout de SQLite que contiene `seed_length`, `source_event_seqs` y `surface_op` es la versión 4 de schema. Los layouts anteriores de la versión 3 eran ambiguos, así que todo `user_version` no actual se rechaza sin migración bajo la política pre-release.

### 3. El replay deriva el script del hijo después del límite

El `parseSessionHeader` de `dsh-llm-replay` ahora también lee `seedLength` (ausente ⇒ 0), y `loadSessionScripts` deriva las entradas de un hijo de `parseSessionLog(text).slice(seedLength)` — los eventos en o después del límite, es decir, las llamadas de modelo propias del hijo. Para un hijo de spawn `seedLength` es 0 y esto es un no-op, así que los escenarios de spawn quedan byte por byte sin cambios.

Esto cierra la brecha de corrección del enrutado, y dos escenarios de fork grabados lo ejercitan de extremo a extremo — véase [Grabar escenarios de instantánea de fork y mixtos spawn+fork](../../archived/testing/2026-06-22-fork-snapshot-scenarios.md).

## Alternativas consideradas

- **Derivar el límite heurísticamente en `llm-replay`** (el prefijo sembrado es un bloque contiguo de eventos del padre que termina en el último `turn/end` antes del primer `user/message` del hijo). Rechazado: una heurística frágil en el harness de pruebas que re-deriva un hecho que el productor ya conoce. Persistir el límite en su origen (el backend de fork) es la regla «explícito > implícito en los límites de paquetes» aplicada a través del límite de persistencia — el lector de un fixture de hijo nunca tiene que reconstruir dónde terminó la herencia.
- **Fijar la versión del formato en lugar de subirla** (la postura «inestable» `SESSION_FORMAT_VERSION = 0` que usa el registro de eventos). Rechazado para el layout de la *tabla* de SQLite: `SCHEMA_VERSION` es la perilla monótona de subir-y-rechazar (un conjunto enumerable pequeño de revisiones que vale la pena distinguir), distinta de la `version` del vocabulario de eventos. Añadir una columna es precisamente el cambio de tabla rompedor que versiona, así que sube.

## Consecuencias

- Un nuevo campo de cabecera persistido en core + ambos backends; el catálogo de subsistemas (`persistence.md`) se actualiza en el mismo cambio (sus bloques `type-equiv` de `SessionHeader` / `CreateSessionOptions`).
- Las bases de datos SQLite existentes en schema v2 se rechazan al abrir (sin datos de usuario pre-release).
- El replay de spawn no cambia (`seedLength` 0). El replay de fork ahora enruta un hijo a su propio script; cubierto por una regresión en los tests de `llm-replay` (un fixture de hijo cuyo prefijo sembrado lleva un chunk del padre — el script hijo derivado debe excluirlo, probado en rojo sin el slice) y una prueba de ida y vuelta de persistencia (ambos backends, vía el contrato compartido del coordinador).
