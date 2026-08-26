# Agent Note: El seam de subagente

Status: implemented

[English](2026-06-21-subagent-capability-seam.md) | Español

> El seam completo está entregado: la interfaz `dsh-subagent` y el Consumer `dsh-tool-subagent`; los dos backends en proceso (`dsh-subagent-spawn-in-process`, `dsh-subagent-fork-in-process`); la infraestructura de instantáneas para agents anidados ([replay de instantáneas por sesión](../testing/2026-06-22-subagent-snapshot-replay.es.md)); y los backends fuera de proceso ACP, Codex y Claude Code ([Agent Note de ACP](2026-06-22-acp-subagent-backend.es.md), [Agent Note de providers de producto](2026-08-04-claude-code-and-codex-subagent-backends.es.md)).

## Problema

El harness arrastra un seam aplazado desde hacía tiempo para los **subagentes** — un agent que delega trabajo en otro agent. La intención quedó esbozada en las interfaces `Agent`/`AgentLoop` ([packages/core/agent/src/types.ts](../../../../packages/core/agent/src/types.ts), [packages/core/agent-loop/src/index.ts](../../../../packages/core/agent-loop/src/index.ts)): una opción de creación que referencia a un agent padre (fork = sembrar la sesión hija con el registro de eventos del padre; spawn = sesión fresca), con el hijo devuelto como un handle `Agent` para que el steering y la suscripción a eventos funcionen de manera uniforme.

**Múltiples implementaciones de subagente deben coexistir en runtime.** Un padre puede querer un hijo barato en proceso para una subtarea acotada Y un hijo aislado fuera de proceso (sobre ACP) en la misma sesión. Los transportes:

- **en proceso** — un `Agent` concreto hijo sobre el mismo `Context` (el más barato, y casi gratuito dado el factory de agents existente);
- **ACP** — actuar como *client* ACP conduciendo otro proceso de agent (que puede ser otra instancia de nosotros mismos);
- **Codex app-server y Claude Code Agent SDK** — los hermanos de un solo disparo actuales que aplican el mismo contrato de provider con nombre a procesos oficiales de producto ([Agent Note de providers de producto](2026-08-04-claude-code-and-codex-subagent-backends.es.md));
- más adelante: **A2A** usando la misma forma fuera de proceso de «arrancar un hijo, enviarle un prompt, liquidar, cancelar».

## Alternativas consideradas

### Por qué no la forma del seam de bash

El seam de bash ([seams de capacidad](../architecture/2026-06-13-capability-seams.es.md)) registra exactamente un `ShellExecutor` por contexto; cargar un segundo lanza una excepción. Eso es correcto para bash (una máquina, una forma de ejecutar un comando) pero incorrecto aquí: la coexistencia es el requisito. Así que el servicio de subagente es un **registro de providers con nombre** — cada implementación se registra bajo un nombre único y el llamador elige uno por nombre — reflejando el **registro de adaptadores LLM** (`LlmRuntime.registerAdapter`), no el ejecutor de bash de servicio único. El seam sigue siendo de tres paquetes (Service Definition / Service Provider / Consumer); solo difiere el eje «una vs. muchas implementaciones».

## Decisión

### La frontera de tres paquetes

Un nuevo grupo de paquetes `packages/subagent/`:

| Paquete | Rol |
|---|---|
| `@deepseek-ai/dsh-subagent` | interfaz: `SubagentRuntime` (`ctx.subagents`), `SubagentProvider`, `SubagentRun`, el vocabulario de petición/resultado/capacidad, los eventos `subagent/*` |
| `@deepseek-ai/dsh-subagent-spawn-in-process` | implementación: un hijo fresco en proceso vía `ctx.agents.create` |
| `@deepseek-ai/dsh-subagent-fork-in-process` | implementación: un hijo en proceso sembrado con una instantánea del registro del padre |
| `@deepseek-ai/dsh-subagent-acp` | implementación: un client ACP que conduce un proceso hijo configurado |
| `@deepseek-ai/dsh-subagent-codex` | implementación: un proceso oficial Codex app-server de un solo disparo |
| `@deepseek-ai/dsh-subagent-claude-code` | implementación: un proceso oficial Claude Code de un solo disparo a través del Agent SDK |
| `@deepseek-ai/dsh-tool-subagent` | Consumer: la herramienta `subagent` orientada al modelo sobre `ctx.subagents` |

### La primitiva: `start` asíncrono → `SubagentRun`

Un provider expone `start(request) → Promise<SubagentRun>`. El cumplimiento publica un hijo y transfiere su handle de ejecución al llamador. El trabajo que falla antes de la publicación rechaza `start()`, mientras que los resultados de prompt, turno, cancelación e infraestructura posteriores a la publicación se liquidan a través de `run.result` sin ocultar el id del hijo. Una sola señal cubre la cancelación antes y después de la publicación; `dispose()` cancela el trabajo restante y espera la quietud. Un start rechazado limpia los recursos no publicados y no emite ningún evento de ciclo de vida, mientras que un fallo de resultado posterior a la publicación cierra el par publicado del ciclo de vida. `start` es neutral al transporte; `spawn` nombra solo el backend fresco en proceso.

### Dos clases de capacidad opcional, descubiertas de dos maneras

- **Funcionalidades en el arranque** (`outputSchema`, `depthLimit`, `toolFilter`, `persona`) viajan en un descriptor estático `provider.capabilities`. El servicio comprueba cada una solicitada ANTES de delegar y **rechaza en voz alta** (`SubagentError('UNSUPPORTED_CAPABILITY')`) si el provider no la tiene — nunca aceptadas y luego ignoradas. Deben comprobarse antes de que exista una ejecución, y por eso no pueden ser métodos de runtime.
- **La creación continuable** es el método opcional `SubagentProvider.prepareContinuable`; la presencia es la capacidad y el estrechamiento de TypeScript es el mecanismo de descubrimiento, de modo que ningún flag separado puede divergir de la implementación. El gestor de continuaciones posee la entrega posterior y la reanudación en frío directamente a través de `AgentHandle`, mientras que el `SubagentRun` de un solo disparo no tiene operación de steering ni de reanudación, según lo refinado por [subagentes continuables](2026-07-28-continuable-subagent-conversations.es.md).

### Fork frente a fresco son backends separados, no un flag

Los hijos frescos y los bifurcados son providers separados, no un flag de petición. `dsh-subagent-spawn-in-process` arranca un hijo aislado; `dsh-subagent-fork-in-process` siembra un prefijo balanceado que contiene solo turnos completados del padre. El turno en vuelo queda excluido porque su llamada de subagente aún no tiene resultado y no puede formar un historial de replay válido.

### Aislamiento del hijo y el registro del padre

Cada subagente en proceso corre en su **propia `Session`** (id propio, linaje `parentSession`), persistida de forma independiente. Los providers remotos ACP y de producto de un solo disparo, en cambio, acuñan un id de ciclo de vida con ámbito de padre y no exponen ningún `Agent` local ni `Session` hija; su estado interno permanece en el proceso remoto. En ambas formas, el registro del padre deja constancia solo del `tool/call` de spawn y su `tool/result` (la salida final del hijo, o un resultado fallido con diagnóstico opcional del provider), mientras que los pasos y llamadas de herramienta del hijo permanecen fuera del registro del padre.

### Recolección síncrona (primer corte)

`dsh-tool-subagent` pasa su señal de ejecución a `start()`, espera el resultado del hijo y disponde la ejecución antes de reportar. Los resultados no completados se convierten en resultados de error en lugar de salida parcial exitosa; presentan el diagnóstico seguro opcional del que es dueña la [decisión de permisos no interactivos](2026-08-15-product-subagent-noninteractive-permissions.es.md) separado del texto parcial del assistant. Los rechazos independientes de resultado y de disposal siguen siendo observables de forma independiente.

### La selección de provider es configuración, no algo orientado al modelo

`dsh-tool-subagent` se liga a exactamente un nombre de provider (`Config.provider`); el modelo ve solo `{ description, prompt }`. Para exponer más de un transporte, carga el plugin de herramienta más de una vez, cada instancia ligada a un provider distinto y a un `toolName` distinto (el registro de herramientas rechaza un nombre duplicado). El *servicio* mantiene el registro multi-provider; la *herramienta* elige uno — el schema no lleva ningún parámetro de provider/tipo.

## Pruebas

Las pruebas de registro y de herramienta sustituyen solo el hijo no determinista por un provider local con guion mientras ejercitan el `SubagentRuntime`, el ciclo de vida, la integración de tareas y la herramienta orientada al modelo reales. Las pruebas de regresión del loader siguen cubriendo las exportaciones de provider y Consumer para el fallo descrito en el [post-mortem 0001](../../../../docs/postmortem/0001-acp-default-export-drops-inject.es.md). Las pruebas de registro cubren la seguridad ante recargas, los nombres duplicados y el rechazo de capacidades en el arranque; los escenarios de agent anidado reproducen sin clave a través del [replay de instantáneas por sesión](../testing/2026-06-22-subagent-snapshot-replay.es.md); los backends en proceso también tienen pruebas unitarias con el loop real y un e2e con clave.

## Consecuencias

- **Recursión.** Sin un límite, un hijo en proceso puede ver la herramienta de delegación y recursar. Los backends en proceso implementan el límite absoluto de profundidad opcional y el `toolFilter` con ámbito de globales vivas; ACP anuncia ambas capacidades apagadas y rechaza tal petición. La [Agent Note de controles de composición del subagente](2026-07-12-subagent-persona-tool-filter-and-depth.es.md) posee su semántica exacta y sus límites de seguridad.
- **Bloquear el turno del padre.** La recolección en primer plano mantiene abierto el paso del padre durante toda la duración del hijo. La delegación en segundo plano usa el runtime compartido `ctx.jobs` y las herramientas genéricas `job_*`, el mismo mecanismo de recolección que bash en segundo plano; el seam de subagente en sí permanece agnóstico de la tarea.
- **Progreso en vivo.** Solo el ciclo de vida y el resultado final afloran; un flujo de actualizaciones por fragmento de hijo→padre queda aplazado con el rediseño del segundo plano.
- **Superficie de client ACP.** Proxyar `fs`/`terminal` desde el hijo ACP de vuelta al padre (un modo de workspace compartido) es trabajo futuro; el backend no anuncia ninguna de las dos capacidades, así que el hijo se autoabastece en su propio proceso.
