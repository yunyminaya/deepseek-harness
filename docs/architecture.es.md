# Arquitectura de DeepSeek Harness

[English](architecture.md) | Español

Lee esto antes de cambiar nada en `packages/`. Asume que conoces Cordis; si no es así, empieza por el [primer](cordis-primer.es.md) o por el [tutorial](cordis-tutorial/index.es.md).

Recomendamos usar un agent para explorar el código base y entender su arquitectura.

## Cordis

[Cordis](cordis-primer.es.md) es el framework sobre el que se asienta dsh: los plugins aportan servicios, eventos tipados y efectos reversibles a un contexto compartido. Cada parte del producto es un plugin, incluidos el adaptador de modelo, el registro de herramientas, el log de sesión y el propio agent loop, de modo que cada parte es reemplazable desde la configuración.

No hay un núcleo privilegiado al que aplicar parches: extiendes dsh montando un plugin junto a los demás, y los registros son efectos que se deshacen cuando su plugin se descarga.

## Profiles y bundles

Un `dsh` en ejecución es un árbol de plugins compuesto en el arranque a partir de capas ordenadas.

Un **profile** es una composición con nombre almacenada en el directorio home de Harness. Lista los bundles que apila, contiene los plugins fuera del árbol que instala y guarda el `cordis.patch.yml` propio del usuario. `web` y `headless` se distribuyen como plantillas.

Un **bundle** es un formato de distribución para filas de configuración de Cordis y el código que montan, de modo que todo lo que inserta sigue siendo parcheable por las capas que tiene encima.

Cada uno se declara en su propio `package.json` bajo un campo `dsh`: `dsh.profile` lista los bundles de un profile, y `dsh.bundle` apunta al archivo de parche de un bundle.

[`dsh-base`](../packages/bundle/base/README.es.md) es la primera capa de todo profile: adaptadores de modelo, herramientas, persistencia, política de sandbox y de aprobación, ajustes, credenciales, telemetría. [`dsh-web-app`](../packages/bundle/web-app/README.es.md) añade la aplicación de navegador; [`dsh-headless`](../packages/bundle/headless/README.es.md) añade un runner de un solo uso sin servidor alguno.

Las capas se aplican a una lista de entradas vacía en este orden: cada bundle en el orden listado del profile, después el `cordis.patch.yml` del profile, luego el del nivel del home y por último cualquier overlay `--patch`. Un parche apunta a una fila por su id y reemplaza toda su configuración, o inserta filas nuevas.

Para ver el árbol que tu máquina arranca realmente:

```sh
dsh --profile web --dump-config
```

Cualquier fila que imprima puede reemplazarse con un parche propio.

La mecánica de composición está en [app-boot](../packages/boot/app-boot/README.es.md#profiles); los campos de configuración están en el [catálogo de config](config-catalog.es.md) generado.

## Paquetes core

Estos son algunos paquetes core que contribuyen al árbol de Cordis.

| Paquete | Posee | Clave `ctx` |
|---|---|---|
| [`core/session`](subsystems/session.es.md) | El log `SessionEvent` de solo añadir y el almacén en memoria | `ctx.sessions` |
| [`core/system-prompt`](subsystems/system-prompt.es.md) | El ensamblaje de secciones de prompt y de schemas de herramientas | `ctx.systemPrompt` |
| [`core/tools`](subsystems/tools.es.md) | El registro de herramientas con ámbito y el pipeline de ejecución con guardas | `ctx.tools` |
| [`core/agent`](subsystems/core.es.md) | La interfaz `Agent`, el registro en vivo y los eventos `agent/*` | `ctx.agents` |
| [`core/agent-loop`](subsystems/core.es.md) | El driver predeterminado que implementa esa interfaz | `ctx.agentLoop` |
| [`core/scope`](subsystems/scope.es.md) | La primitiva de registro con ámbito por agent | librería, sin clave |
| [`llm/llm`](subsystems/llm-streaming.es.md) | El vocabulario de mensajes y de flujos más el seam de adaptador | `ctx.llm` |

## Eventos

Los eventos son los puntos de extensión, y elegir el dominio correcto es la primera decisión en la mayoría de los cambios.

- **Los eventos de sesión** son hechos duraderos anexados al log y difundidos a través de `session/event`. Usa uno cuando el hecho debe sobrevivir a una recarga.
- **Los eventos de agent** (`agent/*`) transportan un `Agent` vivo: inbox, step, status, request, validation, continuation. Usa uno para observar o interceptar trabajo en curso.
- **Los eventos de capacidad** adjuntan política y adaptadores a un seam (`fs/*`, `tools/*`, `telemetry/*`) sin importar el loop.

El [mapa de eventos](event-producer-consumer.es.md) lista los productores y consumidores de cada evento.

## Flujo de turnos

Un **step** es una solicitud de modelo más las herramientas que llama. Un **turno** son cero o más steps: se abre antes de que se reclame su primera entrada y se cierra cuando ya no queda nada pendiente.

```text
turn/start
  claim next-step input plus one queued message
  assemble prompt sections + tool schemas
  -> agent/pre-step                   reject | enter(messages)
     reject, or a first enter rewritten empty -> close the turn with no step
     step/start
     append entered messages as user/message
     derive model history from the log
     agent/request -> llm/stream -> assistant/chunk* -> assistant/message
     tool/call* -> tools/pre-execute -> tools/execute -> tools/post-execute -> tool/result*
     step/end
     tools owe another request, or next-step input arrived -> claim -> next step
  -> agent/turn-stopping
turn/end
```

`turn/*`, `step/*`, `user/message`, `assistant/*` y `tool/*` son eventos de sesión duraderos; el resto son puntos de extensión en vivo en tres dominios. `agent/pre-step`, `agent/request`, `llm/stream` y los tres eventos `tools/*` son waterfalls, cuyos listeners deben llamar a `next()` para delegar; `agent/turn-stopping` es serial y no tiene `next()`.

La entrada llega al driver a través de una sola inbox. Algunos mensajes lo despiertan de inmediato; el contexto inyectado espera en la inbox hasta que otro mensaje lo hace.

`agent/pre-step` decide lo que ve el modelo. Los listeners pueden reescribir los mensajes reclamados o rechazarlos sin más; un primer reclamo rechazado o vacío sigue cerrando un turno duradero que no gastó ningún paso, así que el log registra el intento. Cada step lee las secciones de prompt y los schemas de herramientas que registraron los plugins.

Detalles: el [diagrama de secuencia](agent-lifecycle.es.md), el [pipeline de herramientas](tool-execution-pipeline.es.md) y la [cancelación y recuperación de errores](subsystems/core.es.md#the-agent-handle).

## Log de sesión

El log de sesión es la fuente del contexto que ve el modelo. `deriveMessages()` proyecta el historial de modelo desde él, y los eventos brutos `assistant/chunk` preservan la fidelidad de la reproducción y de la interfaz. El fork, la reanudación, los transcripts, la telemetría y la persistencia se derivan todos de este flujo.

**Lo visible para el modelo equivale a lo registrado.** Cualquier cosa que llegue a una solicitud de modelo debe poder reconstruirse desde el log, y un invariante de runtime lo verifica. Por eso una nueva entrada visible para el modelo exige un nuevo evento de sesión: extiende `SessionEventMap` y renderiza desde el log.

## Seams de capacidad

Un **seam** es una capacidad intercambiable con tres roles: una **Service Definition** que declara la interfaz, un **Service Provider** que la implementa y un **Consumer** que la usa, normalmente una herramienta orientada al modelo. Un paquete puede combinar roles, pero un solo rol no es un seam; añadir una capacidad significa diseñar los tres ([grafo de capacidades](capability-seams.es.md)).

Los seams son la razón por la que un solo cambio de provider cambia todo el producto. Los providers de filesystem y de subprocess comparten un mismo mundo de ejecución, así que apuntarlos a un sandbox remoto mueve Bash, PTY y LSP con ellos, sin bifurcaciones de provider. Los [providers de subagentes](subsystems/subagent.es.md) varían con la misma amplitud detrás de una sola interfaz, desde un agent hijo recién creado hasta un turno delegado en otro producto.

[Agent Teams experimentales](subsystems/agent-team.es.md) es un seam de coordinación privado de adhesión voluntaria en `ctx.agentTeams`, con una plantilla duradera, un tablero de tareas y un buzón construidos sobre subagentes continuables.

## Dónde va el comportamiento nuevo

El comportamiento nuevo se engancha a un punto de extensión documentado. Cambiar el propio loop actualiza este mapa.

| Objetivo | Mecanismo |
|---|---|
| Añadir un provider de modelo | registra su adaptador en `ctx.llm` |
| Añadir una capacidad orientada al modelo | registra en `ctx.tools`; su schema se suma al ensamblaje de prompts |
| Dar a una sesión un conjunto de capacidades distinto | compón un preset de agent; una fila de servicio allí necesita un reino `isolate` |
| Añadir ejecución de shell | registra un backend `ctx.shell`; el local hace spawn a través de `ctx.subprocess` |
| Añadir ejecución de terminal persistente | registra un backend `ctx.terminals` más `dsh-tool-terminal` |
| Añadir un comando humano | registra en `ctx.commands`; despacha sin un turno de modelo |
| Añadir trabajo en segundo plano | registra en `ctx.jobs`; las herramientas `job_*` lo recogen o lo detienen |
| Añadir acceso a filesystem o política | registra un provider `ctx.fs` o escucha eventos `fs/*` |
| Confinar procesos con spawn | usa un backend `ctx.sandbox`; los Consumers envuelven el argv antes de hacer spawn |
| Interceptar una solicitud, una herramienta o un turno | usa su evento `agent/*` o `tools/*`; `agent/turn-stopping` detiene un turno |
| Añadir contexto orientado al modelo | llama a `agent.inject()`; aterriza en la siguiente solicitud admitida |
| Añadir integración de UI o de editor | maneja `ctx.agents` y renderiza desde `session/event` |
| Añadir un nodo de Chat de Web Client | registra un `ConversationNodeDefinition` + renderizador con clave |
| Añadir estado de sesión duradero | extiende `SessionEventMap`; renderiza y reproduce desde el log |
| Generar títulos de sesión | registra el único provider `ctx.sessionTitle` |
| Gestionar un objetivo de la misma sesión | usa `ctx.goals`; continúa mediante `agent/*` |
| Hacer fork de una sesión en vivo | `ctx.sessions.fork(source, boundary?, childSessionId?)` |
| Acotar un registro a un agent | usa el `agent.ctx` de ese agent |

El [recetario de extensiones](cookbook/extension-cookbook.es.md) mapea las funcionalidades a las capacidades e indexa las guías paso a paso de [paquetes](cookbook/adding-a-package.es.md), [herramientas](cookbook/adding-a-tool.es.md), [adaptadores de LLM](cookbook/adding-an-llm-adapter.es.md), [nodos de Chat](cookbook/adding-a-conversation-node.es.md) y [tarjetas de ajustes](cookbook/adding-a-settings-card.es.md).
