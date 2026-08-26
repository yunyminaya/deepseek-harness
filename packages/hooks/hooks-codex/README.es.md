# @deepseek-ai/dsh-hooks-codex

[English](README.md) | Español

Un plugin de cordis que ejecuta el subconjunto soportado de la config de hooks **Codex** existente de un usuario sobre los puntos de intercepción canónicos del harness. La mitad de **dialecto Codex** del subsistema hooks. Las primitivas independientes del dialecto vienen de [`@deepseek-ai/dsh-hook-protocol`](../hook-protocol/README.es.md); este puente es dueño de las cargas útiles con forma Codex, del modo de matcher y del mapeo de decisiones.

Este puente implementa un subconjunto deliberado del protocolo de hooks actual de Codex:

- **Cinco de los diez puntos de hook:** `PreToolUse`, `PostToolUse`, `SessionStart`, `UserPromptSubmit` y `Stop`.
- **Matchers solo regex** (sin ruta rápida literal; el matcher es siempre una regex sin anclar).
- **Cargas útiles de stdin en snake_case** con los extras `turn_id`/`model`, escritas **sin** nueva línea final.
- **Sin inyección de env de plugins Codex y sin sustitución de placeholders en tiempo de config** (el comando recibe igualmente el entorno del ejecutor y se ejecuta a través de su shell).
- **Sin ruta de aprobación ni reescritura pre-herramienta** — un hook puede bloquear, pero el puente no pre-aprueba ni reemplaza la entrada de la herramienta.

Un plugin de cordis nativo podría hacer todo lo que hace este puente, con más potencia; el puente existe solo como camino de compatibilidad para el subconjunto Codex mapeado (véase [el Agent Note de puntos de intercepción](../../../.agents/notes/implemented/feature/2026-06-30-interception-extension-points.md)).

## Config

```ts
import type { Config } from '@deepseek-ai/dsh-hooks-codex'
const config: Config = {
  configPath: '/path/to/.codex/hooks.json', // required
  model: 'deepseek-v4',                      // optional: stamped on every payload (Codex includes `model`)
  defaultTimeoutMs: 600_000,                 // optional: per-hook timeout when a hook sets none
  stderrSummaryMaxChars: 500,                // optional: char cap on the hook/result event's persisted stderr summary
}
```

En un `cordis.yml`:

```yaml
- dsh-hooks-codex:
    configPath: ./.codex/hooks.json
    model: deepseek-v4
```

La config se parsea **una vez** en la carga. `configPath` es **a nivel de proceso** — una ruta relativa se resuelve contra el cwd de lanzamiento del proceso en el momento de la carga, no por sesión (`TODO(per-session-hook-config)`). Un fallo de lectura/parseo queda contenido (registra + no registra nada); un matcher regex inválido en un evento que consume matchers es uno de esos fallos y notifica su patrón y evento. Solo se ejecutan los hooks `type: 'command'` síncronos — un hook que no es de comando o con `async: true` se parsea-y-omite con una advertencia. Un hook acepta `timeout` o el alias `timeoutSec`; uno que no fije ninguno se ejecuta bajo el valor por defecto de referencia del protocolo (`DEFAULT_HOOK_TIMEOUT_MS` de `dsh-hook-protocol`, 10 minutos). Los eventos fuera de los cinco puntos soportados por el puente se descartan en el parseo.

Los hooks en sí se ejecutan en el workspace de sesión del agent: para los puntos con ámbito de agent, el puente pasa el `cwd` de la sesión como directorio de trabajo del proceso de hook, así que un hook opera en el árbol de proyecto del usuario, no en el directorio de lanzamiento del servidor.

## Puntos de hook → Decisions tipadas

| Hook de Codex | Punto del harness | Mapeo |
|---|---|---|
| `SessionStart` | `agent/session-start` (emit) | la salida de un hook de stdout plano → additionalContext → `agent.inject()` |
| `UserPromptSubmit` | `agent/pre-step` (waterfall) | `block` (salida 2) → `PreStepDecision.reject`; solo additionalContext → delega vía `next()` y luego añade un mensaje de fuente separada a una decisión `enter` aguas abajo |
| `PreToolUse` | `tools/pre-execute` (waterfall) | `block` → `PreToolDecision.deny` (sin `allow`/`ask`) |
| `PostToolUse` | `tools/post-execute` (waterfall) | `block` → `block` con retroalimentación; solo additionalContext → delega vía `next()` y luego antepone un contexto de fuente separada a la decisión aguas abajo; Code Mode difiere los contextos de sub-llamadas hasta el resultado del `run_code` exterior |
| `Stop` | `agent/turn-stopping` (serial) | un hook Stop bloqueante alimenta su razón a través de `steer()`, forzando otro paso |

La carga útil de una llamada de herramienta lleva el `tool_name` real (el mismo valor que prueba el matcher) y la forma `tool_input: { command }` de Codex (el argumento `command` cuando está presente, si no `''`). El sujeto del matcher es el nombre de la herramienta (`PreToolUse`/`PostToolUse`) o la fuente de la sesión (`SessionStart`); `UserPromptSubmit`/`Stop` ignoran los matchers.

Cada carga útil de stdin con ámbito de agent lleva `session_id` y `transcript_path`. El puente resuelve esta última a través de `ctx.sessionPersistence.locate(session.header)` cuando está disponible y envía `null` en caso contrario, preservando la forma `string | null` de Codex. La búsqueda no crea ni vacía el artefacto, así que una ruta puede faltar antes del primer checkpoint de fin de turno u omitir el turno abierto actual.

`SessionStart` — el único punto emit — se ejecuta desacoplado; cada cadena de ejecución se rastrea, y eliminar el puente aborta un proceso de hook aún en marcha y luego drena la continuación antes de que dispose resuelva (`createDetachedRuns` en `dsh-hook-protocol`).

## Fuente de contexto

El contexto inyectado lleva una fuente explícita `{ kind: 'plugin', plugin: 'hooks-codex' }` para que el mensaje duradero nunca se confunda con un prompt de usuario.

## Model Experience

### Contexto proporcionado por hooks

#### Lo que ve el modelo

Los hooks `SessionStart`, de prompt aceptado y post-herramienta pueden añadir mensajes de contexto atribuidos a su fuente; un hook `Stop` bloqueante añade su razón como steering del siguiente paso.

#### Efecto de tokens

Sin coste cuando los hooks no devuelven contexto. El texto de los hooks depende de los datos, se registra y se reenvía hasta la compactación.

#### Efecto de KV Cache

Solo añadidura; el contenido recién visible sigue el prefijo de petición reutilizable y no invalida las entradas de caché KV existentes.

### Prompt o resultado de herramienta bloqueados

#### Lo que ve el modelo

Las razones suministradas por el provider pasan verbatim. Cuando faltan, un prompt bloqueado usa exactamente `blocked by UserPromptSubmit hook`, una herramienta denegada se convierte en `Error: blocked by PreToolUse hook`, la retroalimentación post-herramienta bloqueada es exactamente `blocked by PostToolUse hook`, y una parada bloqueante añade steering exactamente `continue: blocked by Stop hook`. El `systemMessage` de Codex no se expone.

#### Efecto de tokens

Bloquear un prompt elimina sus tokens de petición; la denegación o la retroalimentación añaden el texto de respaldo o del provider retenido; la continuación forzada paga otra petición completa.

#### Efecto de KV Cache

Un prompt bloqueado no envía ninguna petición y no invalida nada. La denegación, la retroalimentación y el contexto de continuación forzada se añaden después del prefijo reutilizable sin reescribirlo.

## Limitaciones conocidas y trabajo diferido

- **Eventos de hook no soportados (5 de los 10 actuales de Codex):** `PermissionRequest`, `PreCompact`, `PostCompact`, `SubagentStart` y `SubagentStop`. La config de estos eventos se descarta silenciosamente durante el parseo. La línea base de comparación es la [referencia oficial de hooks](https://learn.chatgpt.com/docs/hooks) de Codex.
- **`SessionStart` es parcial:** funcionan el stdout plano y el `additionalContext` JSON, pero el hook se ejecuta desacoplado, así que el contexto puede perderse la primera petición (`TODO(session-start-gating)`).
- **`UserPromptSubmit` es parcial:** funcionan el bloqueo y el contexto de stdout plano o JSON, pero los controles comunes `systemMessage` y `{"continue": false}` no se aplican.
- **`PreToolUse` es parcial:** el bloqueo funciona, pero `additionalContext`, `permissionDecision: "allow"` y `updatedInput` se ignoran. Cada herramienta se representa como `tool_input: { command }`, así que los argumentos de herramientas que no son de shell no se exponen fielmente al hook.
- **`PostToolUse` es parcial:** funcionan la retroalimentación bloqueante y el `additionalContext` JSON, pero `{"continue": false}` no se aplica, los argumentos de herramientas que no son de shell se reducen a `{ command }`, y la salida de herramienta estructurada se aplana a texto en `tool_response`.
- **`Stop` es parcial:** el bloqueo fuerza otro turno de modelo, pero `stop_hook_active` es siempre `false`, `last_assistant_message` es siempre `null`, y `{"continue": false}` no se aplica. Un hook incondicionalmente bloqueante fuerza por tanto la continuación de cada paso salvo que se autolimite (`TODO(stop-loop-guard)`).
- **Los campos comunes de carga útil y salida son parciales:** cada evento mapeado informa del `model` configurado estáticamente y de `permission_mode: "default"` en lugar de los valores de runtime actuales de Codex. `systemMessage` se registra + avisa pero no se expone, y `{"continue": false}` se registra pero no aplica el comportamiento de parada específico del evento de Codex (`TODO(hook-continue-false)`).
- **La carga de config y la ejecución son parciales:** un único `configPath` a nivel de proceso se parsea en la carga; las capas de usuario, proyecto, sesión, sistema/gestionadas y de plugins activas de Codex, los controles de confianza y la forma de hook `config.toml` en línea no están implementados (`TODO(per-session-hook-config)`). Solo se ejecutan los handlers de `command` síncronos, los metadatos actuales como `statusMessage` y `commandWindows` se ignoran, y los handlers que coinciden se ejecutan en serie en lugar de con la semántica de lanzamiento concurrente de Codex.
