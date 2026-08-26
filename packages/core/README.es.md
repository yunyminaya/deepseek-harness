# core/ — columna vertebral de la API de producto

[English](README.md) | Español

El registro de sesión, el ensamblaje del system-prompt, el registro de herramientas, el vocabulario del agent, la selección de modelo por defecto del despliegue y el loop concreto que forman la columna de control por defecto del harness. Son paquetes **de producto** — la superficie estable contra la que construyen los plugins y los Consumers.

| Paquete | Rol | Clave de ctx |
|---|---|---|
| [`scope/`](scope/README.es.md) | Primitiva de registro de contexto con ámbito | librería — sin clave de ctx |
| [`session/`](session/README.es.md) | Registro de sesión basado en eventos y almacén en memoria | `ctx.sessions` |
| [`system-prompt/`](system-prompt/README.es.md) | Registro de ensamblaje de prompts y schemas de herramientas | `ctx.systemPrompt` |
| [`tools/`](tools/README.es.md) | Registro de herramientas con ámbito y pipeline de ejecución | `ctx.tools` |
| [`agent/`](agent/README.es.md) | Interfaz del Agent, registro y vocabulario de eventos | `ctx.agents` |
| [`agent-default-model/`](agent-default-model/README.es.md) | Selección de modelo por defecto compartida por los puntos de entrada del Agent | `ctx.agentDefaultModel` |
| [`agent-loop/`](agent-loop/README.es.md) | Driver de agent concreto por defecto | `ctx.agentLoop` |

`scope` suministra la primitiva de ámbito compartida. `agent` es dueño del contrato público, mientras que `agent-loop` es su implementación por defecto; los plugins de extensión dependen del seam para que el driver siga siendo intercambiable. `agent-default-model` es dueño de la selección de despliegue que un punto de entrada del Agent usa solo cuando una sesión no tiene selección propia.

Las composiciones ejecutables pertenecen a [`examples/agent-spine-demo`](../examples/agent-spine-demo/README.md); este grupo solo posee las piezas intercambiables de la columna.

La referencia de subsistema — el mapa del loop paquete por paquete, el identificador `Agent` y sus contratos de entrega e interceptación — está en [docs/subsystems/core.md](../../docs/subsystems/core.es.md); la composición ejecutable por defecto es [`examples/agent-spine-demo`](../examples/agent-spine-demo/README.md).
