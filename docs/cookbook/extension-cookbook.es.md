# Recetario: formas de los plugins de extensión

[English](extension-cookbook.md) | Español

Patrones de referencia para las extensiones del harness. Los fragmentos omiten los imports y las implementaciones auxiliares y no están completos para copiar y pegar. Para rutas concretas de autoría, consulta la [lista de verificación de paquetes](adding-a-package.es.md), el [tutorial de la primera herramienta](../user/develop/basic/tool.es.md), la [referencia de herramientas](adding-a-tool.es.md) y la [guía de adaptadores de LLM (modelo de lenguaje de gran tamaño)](adding-an-llm-adapter.es.md); la [arquitectura](../architecture.es.md) es la dueña del mapa del sistema y de los puntos de extensión.

## Un plugin de herramienta

Una herramienta se registra en `ctx.tools`. El ejemplo anotado de `defineTool` (argumentos tipados de `execute`, construcción de resultados, el patrón `run_in_background`) vive en [adding-a-tool.md](adding-a-tool.es.md): esa guía es la fuente de verdad para las definiciones de herramientas. `ctx.tools.register()` también acepta directamente `ToolDefinition`s de JSON Schema en bruto (así llegan las herramientas provenientes de MCP); `defineTool` es el helper tipado para las herramientas propias (first-party).

## Un plugin de hook (ejemplo de puerta de permisos)

Esta puerta de permisos es un ejemplo de plugin de hook. Devuelve una decisión tipada desde la puerta `tools/pre-execute` para permitir o denegar una llamada; los plugins de sandbox, de permiso y de modo plan pueden usar este punto de extensión. Los plugins de hook pueden interceptar otros puntos de extensión y no son puertas de permisos por naturaleza. Un «hook nativo» es un plugin de Cordis ordinario en un punto de intercepción; no necesita ningún protocolo externo.

```ts
import type { Context } from '@deepseek-ai/cordis'
import type { PreToolDecision, ToolExecution } from '@deepseek-ai/dsh-tools'

declare function isAllowed(exec: ToolExecution): Promise<boolean>

export const name = 'permission-gate'

export function apply(ctx: Context) {
  ctx.on('tools/pre-execute', async (exec, next): Promise<PreToolDecision> => {
    if (!(await isAllowed(exec))) {
      return { kind: 'deny', reason: 'Denied by policy.' }
    }
    return next()
  })
}
```

Este waterfall (cascada de eventos) es la capa de política reordenable. Usa `ctx.tools.guard()` cuando una invariante necesita una denegación final monótona, `tools/execute` cuando un plugin debe envolver la vida útil real del dispatch (timeouts/reintentos/métricas; solo `exec.signal` es sustituible), `tools/post-execute` para la transformación explícita de resultados y `tools/result` para la observación contenida del resultado final inmutable. La [guía adding-a-tool](adding-a-tool.es.md#execution-policy-and-observation) da la regla de selección.

## Un plugin de UI

Un plugin de UI renderiza desde el flujo `session/event` (el flujo de tokens del asistente como `assistant/chunk`, más los límites de turno/paso y la actividad de herramientas) y devuelve la entrada mediante `agent.followup()` / `agent.steer()`. Un plugin de navegador que aporta una fila de negocio al Web Client integrado registra en su lugar un `ConversationNodeDefinition` y un renderizador de Chat con clave; sigue la [guía de Conversation Node](adding-a-conversation-node.es.md).

```ts
import type { Context } from '@deepseek-ai/cordis'
import { createUserMessage } from '@deepseek-ai/dsh-llm'
import { SessionId } from '@deepseek-ai/dsh-session'

declare function render(text: string): void
declare function onUserInput(handler: (text: string) => void): void

export const name = 'my-ui'
export const inject = ['agents']

export function apply(ctx: Context) {
  ctx.on('session/event', (_session, event) => {
    if (event.type === 'assistant/chunk' && event.data.chunk.type === 'text-delta') {
      render(event.data.chunk.text)
    }
  })
  onUserInput(text => ctx.agents.get(SessionId('client-session'))?.followup(createUserMessage({
    content: [{ type: 'text', text }],
    source: { kind: 'user' },
  })))
}
```

## Un driver de protocolo externo

Un *driver de protocolo* adapta un par del wire a `ctx.agents`; puede servir a una UI o a un cliente de automatización. Un driver stdio es dueño de stdout, crea o reanuda agents (agente) mediante la fábrica y asigna las peticiones del protocolo a `followup()` o `cancel()`. Una petición prompt de bajo nivel devuelve su recibo de encolado duradero; no obtiene un resultado correlacionando `MessageId` con `turn/end`. Publica el estado del agent completo por separado. Un método de automatización puede esperar desde su recibo hasta el siguiente idle y resumir ese intervalo que posee explícitamente, mientras que una UI normalmente sigue observando el flujo de eventos abierto. Derriba los agents con `AgentHandle.dispose()` para que la liberación llegue al reposo (quiescence).

[`packages/acp/acp`](../../packages/acp/acp) es el ejemplo resuelto solo de automatización: expone sesiones de texto nuevas sobre Agent Client Protocol JSON-RPC stdio, emite el texto de asistente confirmado y registra un respondedor de permisos de máquina de un solo uso para los agents que posee. Su [README](../../packages/acp/acp/README.es.md) define los métodos exactos, el orden de los eventos y el contrato de ciclo de vida.

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'my-protocol-bridge'
export const inject = ['agents', 'sessions', 'sessionPersistence']

export function apply(ctx: Context) {
  // Stream every logged assistant text/reasoning delta out to the client.
  ctx.on('session/event', (_session, event) => {
    if (event.type === 'assistant/chunk') {
      const chunk = event.data.chunk
      if (chunk.type === 'text-delta') {
        // sendToClient({ kind: 'message_chunk', text: chunk.text })
      }
    }
  })
  // Inbound "prompt": create/resume an agent, feed it, and return its enqueue receipt.
  // Whole-agent status is a separate notification; no turn end belongs to this prompt.
  // Teardown reaches quiescence via AgentHandle.dispose() (stop + await exit).
}
```

## Cableados ejecutables

Las hojas (leaves) ejecutables cargan sus árboles de plugins desde `examples/*/cordis.yml`; los scripts raíz `demo:*` y esos directorios hoja son el inventario autoritativo. El lanzador `dsh` del producto es dueño de la ejecución Web y headless de un solo uso; las hojas ACP usan [`@deepseek-ai/dsh-acp-demo`](../../packages/examples/acp-demo) y las hojas JSON-RPC usan [`@deepseek-ai/dsh-sdk-jsonrpc-demo`](../../packages/examples/jsonrpc-demo). La hoja headless de instantánea monta [`@deepseek-ai/dsh-agent-spine-demo`](../../packages/examples/agent-spine-demo) y la persistencia JSONL de forma explícita, y luego las impulsa mediante un fixture de prueba propiedad del ejemplo en lugar de un paquete de aplicación publicado.

## El mapa funcionalidad → mecanismo

Cada funcionalidad del producto se asigna a un listener en un punto de extensión documentado — la afirmación del microkernel hecha comprobable ([Agent Note del microkernel](../../.agents/notes/implemented/architecture/2026-06-11-microkernel-event-taxonomy.es.md)). Ninguna fila modifica el loop.

`system-prompt/assemble` es una transformación cooperativa experta de ensamblado completo: el ensamblado que devuelve es autoritativo, por lo que quienes escriben listeners son dueños de conservar las contribuciones activas del Code Mode y del protocolo de salida estructurada. Prefiere `ctx.tools.restrict()` para el filtrado de herramientas que debe mantenerse alineado entre la presentación, la búsqueda y la ejecución.

| Funcionalidad del producto | Mecanismo del plugin |
|---|---|
| Sistema de hooks (nivel de usuario + proyecto) | listeners en `agent/session-start`, `agent/pre-step`, `agent/request`, `tools/pre-execute`, `tools/post-execute` y `agent/turn-stopping`; los waterfall devuelven decisiones tipadas, mientras que `agent/turn-stopping` puede hacer steering de otro paso; los puentes `dsh-hooks-claude-code` / `dsh-hooks-codex` asignan los archivos de configuración de hooks a estos puntos de extensión |
| `/goal` | `ctx.goals` es dueño del estado duradero, `dsh-goal-round-driver` programa rondas dentro de la misma sesión a través del `Agent` público, y productores de comando/herramienta separados exponen el control humano/de modelo |
| `/loop` | en el evento de sesión `turn/end`, haz `followup()` de la siguiente iteración; o fuerza la continuación (force-continue) |
| Flujo de trabajo dinámico | `ctx.workflowEngine` + el motor de hilo de trabajo + la herramienta `workflow`; los hijos estructurados en proceso hacen cumplir la salida con registros de prompt/herramienta con ámbito, una guard de herramienta monótona, el commit final de `tools/result` (incluido el `run_code` envolvente) y el marcador monótono `concludeTurn()` de la ejecución de salida estructurada |
| Mensajes en cola + de steering | `Agent.followup()` / `Agent.steer()` del núcleo |
| Compactación de contexto (automática + manual) | el seam `ctx.compaction` + `dsh-compaction-basic`; la presión automática se ejecuta en `agent/pre-step` en serie, la recuperación canónica de desbordamiento se ejecuta en `agent/request-error`, y los llamadores manuales usan el mismo servicio de compactación ([Agent Note de compactación](../../.agents/notes/implemented/feature/2026-06-18-compaction-capability-seam.es.md)) |
| Configurabilidad del prompt del sistema | `ctx.systemPrompt.section()` con ordenación y sombreado local al ámbito |
| AGENTS.md (raíz) | un provider de secciones que lee el archivo |
| AGENTS.md (subdirectorio, al tocar) + avisos de cambio de archivo | `agent.inject()` desde un watcher / listener de resultado de herramienta |
| Herramientas integradas | `ctx.tools.register()`; los schemas fluyen al ensamblado automáticamente — las familias `dsh-tool-*` (bash, fs, web, subagent, todo) son los ejemplos publicados |
| ToolSearch / divulgación progresiva | sustituye un registro con ámbito de `ctx.tools.restrict()` cuando cambia el conjunto visible; el registro mantiene alineadas la presentación, la búsqueda y la ejecución |
| Fecha límite / reintento / métricas de herramientas | envuelve el dispatch del núcleo con `tools/execute`; un wrapper puede sustituir `exec.signal`, delegar e inspeccionar el resultado normalizado en una única vida léxica |
| Métricas / auditoría / captura finales del resultado de herramienta | observa los resultados autoritativos e inmutables con `tools/result`; usa `tools/post-execute` solo cuando el plugin deba transformar el resultado o adjuntar contexto |
| Política de turno terminal monótona | llama a `ToolExecution.concludeTurn()` desde la herramienta terminal que tuvo éxito; las llamadas de herramienta posteriores de la misma respuesta siguen pudiendo bloquearse con la guard, y el loop se detiene después del paso |
| Sandbox de subprocesos (landlock / sandbox-exec) | usa un backend `ctx.sandbox` mediante `dsh-bash-sandbox`; usa `tools/pre-execute` para la denegación a nivel de capacidad |
| Sistema de permisos / AskUserQuestion | devuelve `ask` desde `tools/pre-execute` y responde a través de `ctx.approval`; registra una herramienta ask separada orientada al modelo para las preguntas ordinarias del usuario |
| Modo plan | [`@deepseek-ai/dsh-plan-mode`](../../packages/plan/plan-mode/README.es.md) — estado `plan/mode` registrado, la sección de orientación `plan:policy`, la entrada `/plan [message]`, la salida directa `/plan off` y la salida `exit_plan_mode` revisada por el usuario; el cumplimiento se mantiene en los ejes independientes de sandbox/aprobación |
| Delegación a subagentes | el registro de providers `ctx.subagents` (`dsh-subagent-spawn-in-process`/`-fork`/`-acp`/`-codex`/`-claude-code`/`-dsh-sdk`) + `dsh-tool-subagent` que expone un provider configurado al modelo |
| MCP | un plugin por servidor: descubre las herramientas → `ctx.tools.register()` |
| Skills | registro de sección + herramienta; haz `inject()` del contenido del skill al invocarlo |
| Memoria | provider de sección + herramienta |
| Tareas programadas (cron) | un plugin registra herramientas de programación invocables por el modelo; el temporizador se dispara → `followup(…, {source: {kind: 'cron', …}})` cuando está idle / notificación `inject()` cuando está ocupado |
| UI (GUI; el CLI emite JSONL) | escucha `session/event` (chunks del asistente, límites, actividad de herramientas); entrada → `followup()` |
| Nodo de negocio de Chat del Web Client | registra un `ConversationNodeDefinition` y un renderizador con clave `conversation.chat.node` |
| SessionTelemetryBackend / traza reproducible | `session/event` → JSONL; la reproducción (replay) = `sessions.create(id, { seed })` |
| Adaptadores de modelo | subclase de `LlmAdapter` mediante `registerAdapter` (`dsh-llm-deepseek`, `dsh-llm-pi-ai`) |
| Recarga en caliente de plugins | cada registro es un `ctx.effect` → el HMR (hot module replacement) vendored funciona sin más |
