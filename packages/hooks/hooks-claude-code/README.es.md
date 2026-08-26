# @deepseek-ai/dsh-hooks-claude-code

[English](README.md) | Español

Un plugin de cordis que ejecuta el subconjunto soportado de hooks de comando de la config de hooks **Claude Code** existente de un usuario (un `hooks.json`, o la clave `hooks` de un archivo de ajustes) sobre los puntos de intercepción canónicos del harness. Es la mitad de **dialecto CC** del subsistema hooks: es dueño de las cargas útiles de stdin por evento con forma CC del puente, la sustitución de env + `${CLAUDE_PLUGIN_ROOT}`/`${CLAUDE_PROJECT_DIR}` de CC, y el mapeo del resultado neutro de un hook sobre las Decisions tipadas del harness. Las primitivas independientes del dialecto (matcher, codec de código de salida/stdout, ejecución `ctx.shell`, fusión más restrictiva, los eventos `hook/*`) vienen de [`@deepseek-ai/dsh-hook-protocol`](../hook-protocol/README.es.md).

Un plugin de cordis nativo podría hacer todo lo que hace este puente — con más potencia, con retornos tipados y sin frontera de serialización. **El puente existe solo como camino de compatibilidad para el subconjunto de hooks de comando CC mapeado**; cualquier cosa a medida debería ser un plugin nativo sobre los mismos puntos de extensión (véase [el Agent Note de puntos de intercepción](../../../.agents/notes/implemented/feature/2026-06-30-interception-extension-points.md)).

## Config

```ts
import type { Config } from '@deepseek-ai/dsh-hooks-claude-code'
const config: Config = {
  configPath: '/path/to/hooks.json', // required: a hooks.json or a settings file with a `hooks` key
  pluginRoot: '/path/to/plugin',     // optional: replaces ${CLAUDE_PLUGIN_ROOT} in command strings
  projectDir: '/path/to/project',    // optional: replaces ${CLAUDE_PROJECT_DIR} AND sets the hook env var; defaults to the session cwd when omitted
  defaultTimeoutMs: 600_000,         // optional: per-hook timeout when a hook sets none (CC default)
  stderrSummaryMaxChars: 500,        // optional: char cap on the hook/result event's persisted stderr summary
}
```

En un `cordis.yml`:

```yaml
- dsh-hooks-claude-code:
    configPath: ./.claude/hooks.json
    pluginRoot: ./.claude/plugins/my-plugin
    projectDir: .
```

La config se parsea **una vez** en la carga. `configPath` es **a nivel de proceso**: una ruta relativa se resuelve contra el cwd de lanzamiento del proceso en el momento de la carga, así que una sola config se aplica a todo el proceso — todavía no hay descubrimiento de config por sesión (`session/new.cwd`) (`TODO(per-session-hook-config)`). Un fallo de lectura/parseo queda contenido — incluido un matcher regex inválido en un evento que consume matchers, notificado con su patrón y evento — y el puente registra una advertencia y no registra nada en lugar de tumbar el arranque (una ruta mal escrita no debe tumbar al agent). Solo se ejecutan los hooks `type: 'command'` en forma de shell; un hook `http`/`mcp_tool`/`prompt`/`agent` se parsea-y-omite con una advertencia. Un hook sin `timeout` propio se ejecuta bajo el valor por defecto de referencia del protocolo (`DEFAULT_HOOK_TIMEOUT_MS` de `dsh-hook-protocol`, 10 minutos — el valor por defecto de CC).

Los hooks **en sí** se ejecutan en el workspace de sesión del agent: para los puntos con ámbito de agent, el puente pasa el `cwd` de la sesión (el `session/new.cwd`) como directorio de trabajo del proceso de hook, así que el `pwd`/las rutas relativas/los marcadores de un hook operan en el árbol de proyecto del usuario, no en el directorio de lanzamiento del servidor.

## Puntos de hook → Decisions tipadas

| Hook CC | Punto del harness | Mapeo |
|---|---|---|
| `SessionStart` | `agent/session-start` (emit) | additionalContext → `agent.inject()` en la sesión nueva (no puede bloquear) |
| `UserPromptSubmit` | `agent/pre-step` (waterfall) | `deny` → `PreStepDecision.reject`; solo additionalContext → delega vía `next()` y luego añade un mensaje de fuente separada a una decisión `enter` aguas abajo (un listener exterior posterior aún puede rechazar/reescribir) |
| `PreToolUse` | `tools/pre-execute` (waterfall) | `deny` → `PreToolDecision.deny`; `ask` → `PreToolDecision.ask` |
| `PostToolUse` | `tools/post-execute` (waterfall) | `deny` → `block` con retroalimentación; solo additionalContext → delega vía `next()` y luego antepone un contexto de fuente separada a la decisión aguas abajo; Code Mode difiere los contextos de sub-llamadas hasta el resultado del `run_code` exterior |
| `Stop` | `agent/turn-stopping` (serial) | un hook Stop bloqueante alimenta su razón a través de `steer()`, forzando otro paso |
| `SubagentStart` | `subagent/start` (emit) | additionalContext → `agent.inject()` en un hijo en proceso vivo; un hijo remoto no tiene destino de inyección local |
| `SubagentStop` | `subagent/end` (emit) | solo observación |

Los tres puntos emit se ejecutan desacoplados — ningún punto de extensión espera un hook `SessionStart`/`SubagentStart`/`SubagentStop`. Cada cadena de ejecución se rastrea, y eliminar el puente aborta los procesos de hook aún en marcha y luego drena las continuaciones antes de que dispose resuelva (`createDetachedRuns` en `dsh-hook-protocol`).

El sujeto del matcher es el nombre de la herramienta (`PreToolUse`/`PostToolUse`), la fuente de la sesión (`SessionStart`) o una constante `agent_type` de `general-purpose` (`SubagentStart`/`SubagentStop` — el seam de subagente del harness no lleva ninguna etiqueta por tipo, así que el puente informa del valor por defecto propio de la herramienta Task de Claude Code; un matcher `agent_type` por defecto/`*`/vacío dispara, un matcher de tipo específico no); `UserPromptSubmit`/`Stop` ignoran los matchers. Los múltiples hooks configurados en archivo en un punto se ejecutan **en serie, en orden de config**, y se pliegan de forma más restrictiva (`deny > ask > allow`, véase `dsh-hook-protocol`); la ejecución en serie mantiene cada par `hook/invoked`/`hook/result` adyacente en el registro, y el pliegue es independiente del orden para la decisión (véase la nota «ejecutar en serie, no en paralelo» del Agent Note).

Cada carga útil de stdin con ámbito de agent lleva `session_id` y `transcript_path` con forma de string. El puente resuelve esta última a través de `ctx.sessionPersistence.locate(session.header)` cuando está disponible y envía `''` en caso contrario. La búsqueda no crea ni vacía el artefacto, así que una ruta puede faltar antes del primer checkpoint de fin de turno u omitir el turno abierto actual.

## Fuente de contexto

El contexto inyectado lleva una fuente explícita `{ kind: 'plugin', plugin: 'hooks-claude-code' }` para que el mensaje duradero nunca se confunda con un prompt de usuario.

## Model Experience

### Contexto proporcionado por hooks

#### Lo que ve el modelo

Los hooks `SessionStart`, de prompt aceptado, post-herramienta y de inicio de subagente en proceso vivo pueden añadir mensajes de contexto atribuidos a su fuente; un hook `Stop` bloqueante añade su razón como steering del siguiente paso. La inyección en hijos remotos no tiene destino local.

#### Efecto de tokens

Sin coste cuando los hooks no devuelven contexto. El texto de los hooks depende de los datos, se registra y se reenvía en las peticiones de conversación posteriores hasta la compactación.

#### Efecto de KV Cache

Solo añadidura; el contenido recién visible sigue el prefijo de petición reutilizable y no invalida las entradas de caché KV existentes.

### Prompt o resultado de herramienta bloqueados

#### Lo que ve el modelo

Las razones suministradas por el provider pasan verbatim. Cuando faltan, un prompt bloqueado usa exactamente `blocked by UserPromptSubmit hook`, una herramienta denegada se convierte en `Error: blocked by PreToolUse hook`, la retroalimentación post-herramienta bloqueada es exactamente `blocked by PostToolUse hook`, y una parada bloqueante añade steering exactamente `continue: blocked by Stop hook`. `systemMessage` y `updatedInput` se registran o avisan pero no son visibles para el modelo en esta implementación.

#### Efecto de tokens

Bloquear un prompt elimina los tokens de petición de ese prompt; la denegación o la retroalimentación añaden el texto de respaldo o del provider retenido; la continuación forzada paga otra petición completa.

#### Efecto de KV Cache

Un prompt bloqueado no envía ninguna petición y no invalida nada. La denegación, la retroalimentación y el contexto de continuación forzada se añaden después del prefijo reutilizable sin reescribirlo.

## Limitaciones conocidas y trabajo diferido

- **Eventos de hook no soportados (23 de los 30 actuales de Claude Code):** `Setup`, `InstructionsLoaded`, `UserPromptExpansion`, `MessageDisplay`, `PermissionRequest`, `PostToolUseFailure`, `PostToolBatch`, `PermissionDenied`, `Notification`, `TaskCreated`, `TaskCompleted`, `StopFailure`, `TeammateIdle`, `ConfigChange`, `CwdChanged`, `FileChanged`, `WorktreeCreate`, `WorktreeRemove`, `PreCompact`, `PostCompact`, `SessionEnd`, `Elicitation` y `ElicitationResult`. La config de estos eventos se ignora antes del parseo de grupos, así que un evento no soportado no puede invalidar ni registrar hooks. La línea base de comparación es la [referencia oficial de eventos de hook](https://code.claude.com/docs/en/hooks#hook-events) de Claude Code.
- **`SessionStart` es parcial:** se consume el `additionalContext` JSON, pero el contexto de stdout plano, `initialUserMessage`, `sessionTitle`, `watchPaths`, `reloadSkills` y `CLAUDE_ENV_FILE` no se soportan. El hook se ejecuta desacoplado, así que el contexto puede perderse la primera petición (`TODO(session-start-gating)`), y la carga útil omite campos opcionales actuales como `model`, `agent_type` y `session_title`.
- **`UserPromptSubmit` es parcial:** funcionan el bloqueo y el `additionalContext` JSON, pero el contexto de stdout plano, `sessionTitle` y `suppressOriginalPrompt` no se soportan. Salvo que se anule, el puente usa además su valor por defecto de 600 segundos en lugar del timeout de comando específico del evento de 30 segundos de Claude Code.
- **`PreToolUse` es parcial:** funcionan las decisiones `deny` y `ask`; `allow` no pre-aprueba, `defer` no se soporta, `additionalContext` se ignora, y `updatedInput` se registra + avisa pero no se honra ([el Agent Note pre-tool-input-rewrite](../../../.agents/notes/proposed/feature/2026-06-30-pre-tool-input-rewrite.md)).
- **`PostToolUse` es parcial:** funcionan la retroalimentación bloqueante y el `additionalContext` JSON, pero `updatedToolOutput` y `updatedMCPToolOutput` no se soportan y `tool_response` se aplana a texto.
- **`SubagentStart` y `SubagentStop` son parciales:** ambos informan de una constante `agent_type` de `general-purpose` y usan el id de sesión del hijo donde Claude Code informa de la sesión del padre. El contexto de inicio es de mejor esfuerzo y solo puede alcanzar a un hijo en proceso vivo, mientras que el de parada es de solo observación y no puede bloquear al subagente ni alimentarlo con contexto. El inicio omite `transcript_path`; la parada omite además `agent_transcript_path`, `last_assistant_message`, `background_tasks` y `session_crons` y siempre informa de `stop_hook_active: false`.
- **`Stop` es parcial:** el bloqueo fuerza otro turno de modelo, pero `stop_hook_active` es siempre `false`, se omiten `last_assistant_message`, `background_tasks` y `session_crons`, y el tope de bloqueos consecutivos no está implementado (`TODO(stop-loop-guard)`). Un hook incondicionalmente bloqueante fuerza por tanto la continuación de cada paso salvo que se autolimite.
- **Los campos comunes de carga útil y salida son parciales:** las cargas útiles de los eventos mapeados omiten `prompt_id`, `transcript_path`, `permission_mode` y `effort` donde Claude Code los proporcionaría. `systemMessage` se registra + avisa pero no se expone; `{"continue": false}` se registra pero no detiene la ejecución; `suppressOutput`, `stopReason` y `terminalSequence` no se aplican (`TODO(hook-continue-false)`).
- **El soporte de handlers y config es parcial:** solo se ejecutan los handlers de comando en forma de shell. Los handlers `http`, `mcp_tool`, `prompt` y `agent` se omiten; las opciones de handlers de comando como `args`, `async`, `asyncRewake`, `shell`, `if`, `once` y `statusMessage` no se honran. Los handlers que coinciden se ejecutan en serie y no se deduplican, mientras que Claude Code los ejecuta en paralelo y deduplica los handlers idénticos. Un único `configPath` a nivel de proceso se parsea una vez en la carga; el descubrimiento en capas de proyecto, usuario, plugin y política de Claude Code y su recarga en vivo no están implementados (`TODO(per-session-hook-config)`).
