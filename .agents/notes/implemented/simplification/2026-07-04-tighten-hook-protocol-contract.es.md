# Agent Note: Endurecer el contrato del protocolo de hooks — dialecto, campos descartados, dobles defaults y semántica de `hook/result` propiedad de la lib

Status: implemented

[English](2026-07-04-tighten-hook-protocol-contract.md) | Español

## Problema

Cuatro piezas del contrato `dsh-hook-protocol`/bridge no cumplían la disciplina que registra el [Agent Note subagent-observe-enrich](../../archived/feature/2026-06-30-subagent-observe-enrich.md) — este eliminó un campo de ciclo de vida `agentType` por carecer de consumidor, y estas cuatro no superaron la misma prueba:

1. **La variante `'native'` de `HookDialect`** (`packages/hooks/hook-protocol/src/types.ts`) no tenía productores — los bridges estampan `'claude'` y `'codex'`; el único constructor de `'native'` en todo el código era el propio test unitario de la lib. El JSDoc del propio campo define `dialect` como «el bridge que lo ejecutó», y native no es un bridge: el [Agent Note de puntos de extensión de intercepción](../feature/2026-06-30-interception-extension-points.es.md) registra que los hooks nativos no son un paquete y que «un plugin nativo ya puede usar las Decisions tipadas» sin el log durable de hooks, y el ejemplo práctico insignia de plugin nativo afirma exactamente eso (sin eventos `hook/*` en absoluto).
2. **`HookOutput.suppressOutput`** (mismo archivo) era parseado por el codec y descartado en todas las vías: sin rama de bridge, sin fold de merge, sin warn, sin fila en la lista de diferidos — de forma única entre sus hermanos parseados-pero-no-honrados, cada uno de los cuales lleva un diferimiento declarado (`updatedInput` → un warn registrado más la [propuesta pre-tool-input-rewrite](../../proposed/feature/2026-06-30-pre-tool-input-rewrite.es.md); `systemMessage` → un warn registrado más una fila de diferidos en el README; `continue`/`stopReason` → un ancla `TODO(hook-continue-false)` más el registro de decisión `'stop'`). Estructuralmente no hay nada que suprimir: el stdout del hook nunca entra en ningún transcript (el contexto fluye solo vía `additionalContext`; el log registra únicamente `decision`/`stderrSummary`), así que un autor de hook que fijara `suppressOutput: true` obtenía silenciosamente nada, sin warn.
3. **`defaultTimeoutMs` tenía doble default en ambas configuraciones de bridge con un literal flotante** — un `.default(600_000)` de schema Y un fallback `?? 600_000` (`packages/hooks/hooks-claude-code/src/index.ts`, `packages/hooks/hooks-codex/src/index.ts`), dos hogares por bridge para una constante de nivel de protocolo, de modo que los bridges podían desviarse silenciosamente entre sí en el default compartido. *El knob sigue siendo config explícita propiedad del bridge según la regla de no-tunables-hardcodeados (con `stderrSummaryMaxChars` a su lado); el arreglo es el hogar del literal.*
4. **La semántica de `hook/result` vivía en los bridges, dos veces, no en la lib que es propietaria del evento.** `summarize()` — la regla de truncado de stderr — era byte idéntica en `packages/hooks/hooks-claude-code/src/index.ts` y `packages/hooks/hooks-codex/src/index.ts`, y también lo era la regla de cadena de decisión `output.decision ?? (output.continue === false ? 'stop' : 'pass')`; sin embargo, `dsh-hook-protocol` declaraba `hook/result`, documentaba `stderrSummary` como «truncado» sin ser propietario del truncado, y documentaba los valores de decisión sin ser propietario del mapeo. Si un bridge se desviaba (otro tope, otro fallback), la semántica del evento durable compartido se bifurcaría en silencio.

## Decisión

`HookDialect` es el conjunto cerrado de bridges, `'claude' | 'codex'`; `HookOutput` omite el no soportado `suppressOutput`. `hook/result.durationMs` sigue siendo temporización de auditoría durable y solo se normaliza en las instantáneas. Los defaults de referencia viven una sola vez en `DEFAULT_HOOK_TIMEOUT_MS` y `DEFAULT_STDERR_SUMMARY_MAX_CHARS`. `HookResultRecord` y `appendHookResult` son propietarios del resumen de stderr y de la derivación de decisión para ambos bridges. `BLOCKING_EXIT_CODE` es interno al codec.

## Alternativas consideradas

### ¿Por qué no conservarlos?

El vocabulario no soportado puede volver cuando exista un consumidor real. `durationMs` permanece porque la temporización de auditoría durable es útil con independencia de un lector actual. La construcción de payload específica del bridge se queda en cada bridge, mientras que la normalización del evento durable compartido pertenece a la librería de protocolo.

## Verificación

`HookDialect` contiene solo Claude y Codex, y `suppressOutput` está ausente de la fuente, de la documentación de campos parseados y de la normalización. `durationMs` permanece en los eventos y fixtures con saneado de replay. Los defaults `600_000` y `500` viven cada uno una sola vez en la librería de protocolo, las anulaciones de timeout por hook siguen aplicándose, y ambas suites de bridges ejercitan el truncado de stderr y las reglas de decisión propiedad de la librería.

## Consecuencias

Los cambios de `dialect`, `suppressOutput`, tunables y semántica son invisibles en el cable y en las salidas esperadas. El coste fue el churn en `dsh-hook-protocol` y en ambos bridges — barato bajo la postura pre-release, y más barato que dejar que dos copias de la semántica de un evento durable envejezcan separadas.
