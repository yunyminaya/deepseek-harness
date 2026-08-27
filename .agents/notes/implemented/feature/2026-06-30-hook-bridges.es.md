# Agent Note: dsh-hooks-claude-code + dsh-hooks-codex — los puentes de hooks de Claude Code / Codex

Status: implemented

[English](2026-06-30-hook-bridges.md) | Español

## Problema

La superficie de extensión del harness son sus puntos de intercepción tipados ([la Agent Note de puntos de extensión de intercepción](2026-06-30-interception-extension-points.es.md)): un «hook nativo» no es más que un plugin cordis ordinario suscrito a `agent/session-start`, `agent/pre-step`, `tools/pre-execute`, `tools/post-execute`, `agent/turn-stopping`, `subagent/start` o `subagent/end`. Pero los usuarios llegan con configuraciones de hooks de Claude Code (CC) y Codex **ya existentes** — un `hooks.json` (o la clave `hooks` de un archivo de ajustes) lleno de hooks de comandos shell — y quieren que esas se ejecuten sin modificar. Esta Agent Note introduce los dos **plugins puente** que traducen ese protocolo externo de hooks shell hacia los puntos de extensión tipados, construidos sobre la librería compartida del protocolo de conexión ([la Agent Note de la librería de protocolo de hooks](2026-06-30-hook-protocol-lib.es.md)).

La regla central es: **un puente es un adaptador de compatibilidad, no una herramienta potente.** Cualquier cosa que un puente hace (bloquear una herramienta, inyectar contexto, forzar continuación, observar un subagente) un plugin cordis nativo la hace con más potencia — retornos tipados, `ctx` completo, sin frontera de serialización. La razón de ser del puente es ejecutar el subconjunto explícitamente soportado de los hooks de comandos externos de CC/Codex. Eso mantiene cada puente delgado: parsear la configuración, elegir un modo de matcher, construir el payload por evento, llamar a `runHook` + `mergeHookOutputs` de la librería compartida y mapear el desenlace neutral a una Decisión tipada. Los README de los paquetes poseen el inventario exacto y vigente de eventos no soportados y campos parciales frente a los protocolos oficiales.

## Decisión

Dos plugins independientes en el grupo `packages/hooks/`, cada uno un plugin de función/namespace (`name`/`inject`/`Config`/`apply`, SIN export default — véase el [post-mortem 0001](../../../../docs/postmortem/0001-acp-default-export-drops-inject.es.md)) que inyecta únicamente `bash`:

- **`dsh-hooks-claude-code`** — el dialecto CC. Siete de los puntos de hook actuales de Claude Code: `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `Stop`, `SubagentStart` y `SubagentStop`. Posee los payloads por evento con forma CC para stdin (una base de `session_id`/`transcript_path`/`cwd`/`hook_event_name` más campos por evento), `CLAUDE_PROJECT_DIR` más la sustitución de `${CLAUDE_PLUGIN_ROOT}`/`${CLAUDE_PROJECT_DIR}`, y el modo de matcher literal-o-regex. `transcript_path` es el resultado del localizador de persistencia o `''`; stdin lleva un **salto de línea final**.
- **`dsh-hooks-codex`** — cinco de los puntos de hook actuales de Codex: `PreToolUse`, `PostToolUse`, `SessionStart`, `UserPromptSubmit` y `Stop`. Usa un matcher siempre-regex, payloads con forma Codex en snake_case con extras `turn_id`/`model`/`permission_mode` escritos SIN salto de línea final, sin inyección de entorno de plugin de Codex ni sustitución de placeholders en tiempo de configuración, y sin vía de aprobación o reescritura pre-herramienta. `transcript_path` es el mismo resultado del localizador o `null`; los payloads de herramienta llevan el `tool_name` real en la forma reducida `tool_input: { command }`.

### Mapeo desenlace → Decisión

Cada puente mapea el `MergedHookOutcome` neutral de la librería compartida hacia la Decisión tipada de cada punto de extensión:

| Punto de extensión | CC | Codex |
|---|---|---|
| `agent/session-start` (emit) | additionalContext → `agent.inject()` | salida plain-stdout → additionalContext → `agent.inject()` |
| `agent/pre-step` | `deny`→`reject`; solo-contexto→delegar+plegar en `enter` | `block`→`reject`; solo-contexto→delegar+plegar en `enter` |
| `tools/pre-execute` | `deny`→`deny`; `ask`→`ask` | `block`→`deny` (sin allow/ask) |
| `tools/post-execute` | `deny`→`block`+feedback; solo-contexto→delegar+plegar | igual |
| `agent/turn-stopping` | Stop bloqueante → steering del siguiente paso | igual |
| `subagent/start` (emit) | additionalContext → inyectar en un hijo vivo en proceso; un hijo remoto no tiene destino local de inyección | no soportado por este puente |
| `subagent/end` (emit) | solo observación | no soportado por este puente |

El desenlace `ask` del puente CC es una vía real de permiso, no una decisión terminal del puente: `dsh-tools` lo resuelve a través del [seam de aprobación](2026-07-06-approval-seam.es.md) opcional. Un cliente de automatización ACP puede responder la petición de política de máquina de un solo disparo que posee la sesión y `allowed-once` continúa; sin un ApprovalService o respondedor, la llamada falla cerrada a `deny`.

### El origen del contexto es siempre el plugin (la guarda contra el etiquetado erróneo)

Cada `inject()` de puente y cada entrada de contexto adicional pasan explícitamente `{ kind: 'plugin', plugin: 'hooks-claude-code' | 'hooks-codex' }`. La cobertura unitaria fija el `user/message.source` resultante como el plugin y no como el usuario.

`UserPromptSubmit` corre en el pre-step después de `turn/start`, así que cada invocación escribe su par `hook/invoked` / `hook/result` con ámbito de turno. El rechazo deja la entrada reclamada eliminada, cierra el turno como bloqueado sin ningún paso y retiene el par de hooks como su evidencia durable de decisión. El payload de Codex recibe el `turn_id` de ese turno abierto.

### Añadir contexto no es un veto — delega y luego antepone

Un hook que solo adjunta `additionalContext` (sin block/deny) NO es una decisión que el puente deba retornar por sí solo: retornar `enter` desde un listener de waterfall SIN llamar a `next()` cortocircuita a todo listener posterior de `agent/pre-step` / `tools/post-execute`, de modo que un plugin de política/sandbox registrado después del puente jamás vería el prompt. Cada puente por tanto delega vía `next()` antes de añadir su contexto a una decisión enter aguas abajo. El puente preserva todos los mensajes aguas abajo, mientras que un rechazo de pre-step aguas abajo descarta el lote reclamado entero porque no se abre ningún paso. Las decisiones post-herramienta retienen su semántica independiente de `additionalContexts` ordenados, incluida la aplazación de Code Mode a través del resultado del `run_code` exterior. Solo un `deny`/`block` real del propio hook cortocircuita. Las pruebas afirman que un listener posterior aún puede rechazar un prompt tras un hook solo-contexto y que el prompt retenido y los contextos post-herramienta permanecen separados.

### CLAUDE_PROJECT_DIR toma por defecto el workspace de la sesión

Claude Code siempre exporta `CLAUDE_PROJECT_DIR`, y los hooks comunes sin modificar referencian `$CLAUDE_PROJECT_DIR` para rutas relativas al proyecto. Un `config.projectDir` explícito gana; cuando se omite (el cableado ACP por defecto configura solo `configPath`), el puente asigna por defecto la variable de entorno por ejecución al workspace de la sesión del agent — el mismo `session.header.cwd` en el que el hook ya corre — en lugar de dejarla vacía. Así un hook típico con rutas relativas al proyecto funciona en la configuración por defecto.

### Contención

La configuración se parsea UNA vez en la carga; un fallo de lectura/parseo registra y no registra nada, en lugar de tumbar el arranque (una ruta con errata no debe derribar al agent). Solo los hooks shell de forma `type: 'command'` corren para CC; los handlers `http`, `mcp_tool`, `prompt` y `agent` se parsean y se omiten. Codex ejecuta solo handlers de comando síncronos y omite las entradas `async: true` o que no son de comando. Las rutas de listener de emisión (`session-start`, `subagent/start`) corren desacopladas, con su `inject` contenido en un `.catch` que registra (un inject que lance no debe romper el arranque de la sesión ni el loop).

### Dónde corren los hooks y de dónde viene su configuración

Los hooks corren en el workspace de la sesión del agent, así que las rutas relativas apuntan al proyecto del usuario. `configPath` se resuelve una vez contra el cwd de lanzamiento del proceso y aplica a cada sesión. El descubrimiento por sesión de configuración local al proyecto permanece aplazado bajo `TODO(per-session-hook-config)`.

## Brechas de compatibilidad aplazadas

- **Reescritura de tool-input.** Un `updatedInput` de CC/Codex se registra + advierte, no se honra — la reescritura de entrada es un problema de diseño de consistencia aplazado ([la Agent Note de reescritura de entrada pre-herramienta](../../proposed/feature/2026-06-30-pre-tool-input-rewrite.es.md)), porque los argumentos pre-ejecución los leen la auditoría de `tool/call` + el historial de `assistant/message` + la presentación de herramienta, así que una reescritura honesta es una unidad de diseño, no un campo.
- **Guardia de bucle de Stop** (`TODO(stop-loop-guard)`). Claude Code provee `stop_hook_active` y desactiva un hook tras ocho bloqueos consecutivos; Codex provee `stop_hook_active` pero no documenta un tope equivalente. Ambos puentes reportan siempre `false`, así que un hook Stop que bloquea incondicionalmente fuerza la continuación de cada paso — un autor de hooks debe auto-limitarse hasta que llegue el seguimiento de estado.
- **Hook `continue:false` (alto total).** Un hook puede pedir detener toda la ejecución (CC/Codex `continue:false`); el merge compartido lo pliega en `MergedHookOutcome.stop`/`stopReason`, pero ningún puente actúa sobre ello (`TODO(hook-continue-false)`) — los puntos de intercepción aún no tienen una primitiva de «alto total del agent» (una Decisión bloquea/redirige un punto único, no la ejecución completa). Aplazado junto con el trabajo de la guardia de bucle; las peticiones a mitad de turno registran el alto en `hook/result`, y entre tanto el hook conserva su efecto por punto (decisión/contexto).
- **Descubrimiento de configuración.** La ruta es explícita en `cordis.yml` y a nivel de proceso (véase arriba); el recorrido completo de precedencias multinivel de CC/Codex, el descubrimiento por sesión de configuración local al proyecto y el modelo de confianza/hash no están reimplementados (`TODO(per-session-hook-config)`).
- **El contexto de session-start / subagent-start es de mejor esfuerzo** (`TODO(session-start-gating)`). Ambos hooks corren desacoplados del arranque, así que su contexto se inyecta cuando está listo pero puede perder la primera petición o un hijo de vida corta. Garantizar la entrega en la primera petición requiere un punto de extensión de arranque esperado (awaited).

## Alternativas consideradas

**Ejecución de hooks concurrente por punto.** Los motores de referencia ejecutan concurrentemente los hooks emparejados de un punto y pliegan los resultados. Estos puentes los ejecutan **en serie** (`await` por hook dentro del bucle de emparejamiento) y pliegan con el mismo merge más-restrictivo. Serial es deliberado: para los puntos con ámbito de turno mantiene cada par `hook/invoked`/`hook/result` adyacente y en orden determinista, y el pliegue es independiente del orden para la decisión (`deny > ask > allow`), así que el desenlace coincide. El coste es latencia (el hook *N* espera al hook *N−1*) y que los timeouts por hook no se solapan — aceptable para los recuentos de hooks que las configuraciones reales usan; revisar si alguna configuración llega a ramificar tanto que el tiempo de reloj importe.

## Consecuencias

La semántica de matchers, el manejo de códigos de salida y la precedencia de merge viven en `dsh-hook-protocol`; cada puente solo parsea configuración, construye payloads de dialecto y mapea desenlaces. La cobertura por archivo incluye las ramas de configuración más mapeos de extremo a extremo a través de un loop real, `dsh-bash-local` y scripts shell, mientras que un smoke con Loader real guarda la forma de exportación del paquete. Los plugins nativos se saltan el protocolo de conexión y retornan decisiones tipadas directamente.
