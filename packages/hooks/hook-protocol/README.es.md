# @deepseek-ai/dsh-hook-protocol

[English](README.md) | Español

El **núcleo compartido** del protocolo de cable de hooks de Claude Code / Codex. NO es un plugin de cordis — no registra nada ni inyecta nada. Es una **librería** de primitivas neutras respecto al dialecto que los dos plugins puente (`@deepseek-ai/dsh-hooks-claude-code`, `@deepseek-ai/dsh-hooks-codex`) importan para que ninguno reimplemente las mitades idénticas del protocolo.

Codex reimplementa deliberadamente un *subconjunto* del protocolo de hooks de Claude Code — la misma forma de grupo de matchers de `hooks.json`, el mismo contrato de salida de código de salida/stdout, el mismo modelo de ejecución de hooks de comando. Las partes genuinamente compartidas viven aquí; cada puente es dueño solo de lo que difiere.

## Qué se comparte (aquí) frente a por-dialecto (los puentes)

| Aspecto | Aquí (`dsh-hook-protocol`) | El puente (`dsh-hooks-claude-code` / `-codex`) |
|---|---|---|
| Validación y prueba de matchers | `matcherDiagnostic(pattern, mode)` para diagnósticos en tiempo de parseo; `matchesMatcher(pattern, query, mode)` para el emparejamiento contenido en tiempo de ejecución | elige su `mode` (`claude` = literal-o-regex, `codex` = siempre regex) y rechaza un grupo de config que lleve un diagnóstico |
| Ejecutar un hook | `runHook(bash, hook, opts, now)` — carga útil de stdin + env mediante `ctx.shell`, decodifica | construye la carga útil de stdin **por evento** + el **env** del dialecto |
| Decodificar la salida | `parseHookOutput(exit, stdout, stderr)` → `HookOutput` neutro | mapea el `HookOutput` neutro sobre una Decision tipada específica del punto de extensión |
| Fusionar N hooks | `mergeHookOutputs(outputs)` → `MergedHookOutcome` más restrictivo | — |
| Registro duradero | `appendHookInvoked` / `appendHookResult` (eventos de sesión `hook/*`; el `decision`/`stderrSummary` del resultado derivan del `HookOutput` aquí) | los llama alrededor de cada invocación |
| Quiescencia de ejecuciones desacopladas | `createDetachedRuns()` — rastrea las cadenas de ejecución de disparar-y-olvidar; `drain()` aborta y luego las espera | pasa `signal` a cada `runHook` desacoplado, registra `drain` como su disposer de efectos |

## Primitivas

- **`matcherDiagnostic(matcher, mode)` / `matchesMatcher(matcher, query, mode)`** — coincide-con-todo en ausente/`''`/`'*'`; el modo `claude` trata un patrón puro `[A-Za-z0-9_|]+` como literal (la tubería = alternancia de coincidencia exacta) y cualquier otra cosa como regex; el modo `codex` es siempre una regex sin anclar. Los parsers de los puentes descartan los campos de matcher para los eventos sin sujeto de matcher, y luego usan `matcherDiagnostic` para rechazar una regex consumida inválida con un diagnóstico estable antes de registrar cualquier hook. El predicado en tiempo de ejecución contiene aún un patrón inválido como no-coincidencia, así que un llamador directo de la librería no puede lanzar una excepción dentro del agent loop.
- **`runHook(bash, hook, options, now)`** — exige y reenvía el `options.signal` propiedad del llamador, serializa `options.payload` al stdin del hook (con una nueva línea final si y solo si `options.trailingNewline`), fusiona `options.env` después del saneado de credenciales del ejecutor (la API de plugins de confianza `dsh-shell`), honra el `timeoutSec` del hook (si no, `options.defaultTimeoutMs` — el puente es dueño del valor por defecto, con su config por defecto a la referencia `DEFAULT_HOOK_TIMEOUT_MS` de la librería de 10 minutos), y decodifica el resultado (pasando `options.expectedEventName` al codec). La cancelación llega por tanto al kill del grupo de procesos del ejecutor y a la frontera de join. Nunca lanza: un rechazo del ejecutor (fallo de infraestructura) se convierte en un `HookOutput` con `exitCode: undefined` (un error no bloqueante). `now` se inyecta para duraciones comprobables.
- **`parseHookOutput(exitCode, stdout, stderr, expectedEventName?)`** decodifica el estado de salida y el stdout estructurado. La salida 2 bloquea con stderr; los demás fallos no bloquean. Una decisión de permiso específica del hook que coincida anula la decisión heredada de nivel superior; los discriminadores de evento que no coinciden o faltan suprimen solo los campos específicos del evento. Los campos de nivel superior permanecen independientes del evento, y la salida exitosa no JSON se deja al puente.
- **`mergeHookOutputs(outputs)`** — pliega los resultados de cada hook que coincidió en un punto: precedencia de permisos **deny > ask > allow**, parada pegajosa en el primer `continue:false`, razones de bloqueo unidas con `\n\n`, `additionalContext`/`systemMessages` acumulados en orden.
- **`createDetachedRuns()`** — seguimiento de quiescencia para los puntos con forma de emit, que se ejecutan desacoplados (ningún punto de extensión los espera). El puente rastrea cada cadena de ejecución — la ejecución del hook MÁS su continuación — y registra `drain()` como su disposer de efectos: drain dispara la señal de abort del rastreador (así un proceso de hook aún en marcha se mata mediante `runHook`, no se espera hasta su timeout), y luego resuelve una vez que cada cadena rastreada se ha asentado. Que `fiber.dispose()` resuelva significa por tanto que no queda trabajo de hook desacoplado que pueda dispararse en un contexto ya eliminado ([patrones defensivos](../../../docs/defensive-patterns.es.md): dispose debe alcanzar la quiescencia).

## Eventos de sesión `hook/*`

Fusionados por declaración en `SessionEventMap` (solo registro, como `compaction/*` — NO un `SurfaceEventType`, sin `surfaceOp`): `hook/invoked` (se ejecutó un comando de hook) y `hook/result` (su resultado, emparejados por `handlerId`, con `appendHookResult` como dueño de la regla de decisión). Las cargas útiles y el JSDoc por evento están en el [catálogo de eventos de registro de persistencia generado](../../../docs/persistence-catalog.es.md); `stderrSummary` se trunca a `stderrSummaryMaxChars` del registro (la config del puente, referencia por defecto `DEFAULT_STDERR_SUMMARY_MAX_CHARS` = 500; omitido cuando está vacío).

Los registros de invocación/resultado de hooks deben estar dentro de un turno abierto. `UserPromptSubmit`, `PreToolUse`, `PostToolUse` y `Stop` satisfacen esa relación definida por el propietario por construcción. `SessionStart` se ejecuta antes del turno 1 y no recibe ningún registro `hook/*`; su contexto permitido permanece pendiente en la bandeja de entrada hasta que una entrega de despertar abra un turno — véase el Agent Note de hooks.

## Model Experience

Indirectamente, a través de `dsh-hooks-claude-code` y `dsh-hooks-codex`, que pueden convertir la salida de hook parseada en contexto de prompt, resultados bloqueados o retroalimentación de continuación.

#### Efecto de KV Cache

Sin invalidación directa; el consumidor nombrado es dueño de cualquier cambio del prefijo de petición.

## Limitaciones conocidas y trabajo diferido

- **`HookOutput.updatedInput` se parsea pero no se honra** — la reescritura de entrada es un problema de diseño de consistencia diferido ([el Agent Note pre-tool-input-rewrite](../../../.agents/notes/proposed/feature/2026-06-30-pre-tool-input-rewrite.md)); un puente registra y avisa cuando un hook lo fija. Véase `src/types.ts` para los contratos completos.
