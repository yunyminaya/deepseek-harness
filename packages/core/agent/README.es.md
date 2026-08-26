# dsh-agent

[English](README.md) | Español

Interfaz de agent (agente), registro, ámbito del iniciador local al proceso y vocabulario de eventos `agent/*`. Cada plugin (UI, hooks, orquestadores) programa contra el handle `Agent` definido aquí — no tiene ninguna dependencia del loop, por lo que el loop es intercambiable.

El complemento opcional `@deepseek-ai/dsh-agent/invariant` registra las comprobaciones de transición de estado de agent de este paquete con `ctx.invariants`. El servicio de agent raíz no carga diagnósticos de forma implícita.

## Service: `AgentRegistry` (ctx key: `agents`)

Realiza el seguimiento de los agents vivos y transporta al Agent iniciador durante el trabajo asíncrono del driver, sin importar el paquete concreto del loop.

### Public API

La surface de registro con ámbito: `Agent.ctx` es el contexto de ámbito del agent (`dsh-scope`, clave = el agent) — registra a través de él herramientas/secciones/variables/listeners solo para ese agent, y todo se desenrolla en la eliminación. `agentEvents(ctx, agent)` es el dispatcher fusionado para las operaciones ordinarias sobre el sujeto agent (carrier + sujeto inyectado en un solo movimiento); su modo de notificación invoca a todos los listeners y contiene tanto los throws síncronos como los rechazos de las promesas devueltas. El par de ciclo de vida del registro reutiliza un único carrier de enrutado estable. `assembleContextFor(agent)` construye el contexto de ensamblaje por agent (`agent` + `scope` juntos). `installModelSelection(agentCtx, selection)` toma una instantánea de una selección mutable de provider/model/reasoning-effort durante el ensamblaje del prompt, aplica su provider y su modelo a las variables del prompt, y aplica la selección completa al enrutado de la solicitud para un step; la ausencia de un effort seleccionado limpia un effort heredado para que se apliquen los valores por defecto del adaptador/provider. `CreateAgentOptions.setup(agentCtx)` y `ResumeAgentOptions.setup(agentCtx)` componen el mundo con ámbito de un agent nuevo o reanudado mientras ambos objetos permanecen sin publicar. El setup es código de confianza, solo de composición y del mismo proceso: conduce al agent únicamente después de que la creación se resuelva.

`AgentOptions` suministra la ruta inicial de provider/model y un tope de salida `maxTokens` positivo opcional. El loop concreto resuelve cualquier valor por defecto de adaptador para el modelo exacto, registra el tope efectivo en la cabecera de la solicitud y lo aplica a cada solicitud al modelo de conversación; una opción explícita de Agent gana, mientras que la omisión deja el valor por defecto de la ruta del adaptador o del provider al mando.

- `ctx.agents.register(agent: Agent): () => void` — registra un agent **ya construido**. Se elimina junto con el fiber que lo llama.
- Ciclo de vida ordenado avanzado: `enter(agent, owner): () => void` impone `agent.id === agent.session.id`, realiza la comprobación autoritativa de colisión de IDs e inserta sin anunciar; `owner` registra explícitamente la relación viva agent-creador (o `undefined` para una raíz), con independencia del linaje de sesión duradero. `announce(agent)` emite `agent/created` exactamente una vez. Un detach solicitado de forma síncrona por un listener de creación se difiere hasta que ese dispatch se desenrolla, y cada detach comprueba el objeto de entrada capturado, de modo que una capacidad obsoleta no puede eliminar a un reemplazo posterior del mismo ID. La fábrica asíncrona usa esta separación; los plugins ordinarios usan `register()`.
- `ctx.agents.get(id: SessionId): Agent | undefined`
- `ctx.agents.isOwnedBy(id: SessionId, owner: Agent): boolean` — indica si la entrada viva exacta se creó a través del contexto con ámbito de ese agent padre; la propiedad en tiempo de ejecución es independiente del linaje de sesión duradero.
- `ctx.agents.list(): Agent[]`
- `ctx.agents.roots(): Agent[]` — agents vivos creados sin un contexto de agent propietario; una sesión reanudada con linaje puede seguir siendo una raíz en tiempo de ejecución.

#### Ámbito del Agent iniciador

`AgentLoop` ejecuta la vida completa de cada driver concreto dentro de un límite de iniciador. Los drivers concurrentes permanecen aislados: las continuaciones de un driver hijo transportan al hijo, mientras que la continuación del padre recupera al padre en cuanto `withInitiator()` retorna; el seguimiento del drenaje continúa hasta que se resuelve la Promise del driver hijo. La creación, la carga de persistencia y el setup sin publicar permanecen fuera del límite del hijo, de modo que un setup iniciado por un padre hereda el padre mientras `agentCtx.agent` identifica al hijo explícitamente.

- `ctx.agents.currentInitiator(): Agent | undefined` — lee el iniciador heredado sin exigir que exista uno.
- `ctx.agents.requireInitiator(): Agent` — lo lee o lanza `no initiating agent is active`.
- `ctx.agents.withInitiator(agent, operation)` — ejecuta con un Agent exacto y preserva el valor síncrono o la Promise exactos de la operación.
- `ctx.agents.withoutInitiator(operation)` — oculta un iniciador heredado para trabajo local al proceso no relacionado.

El ámbito transporta el propio `Agent` y es local al proceso. La presencia ambiental no es prueba de vida ni autorización; los campos explícitos de Agent siguen siendo autoritativos en los límites de servicio, worker, proceso, persistencia y wire. El teardown rechaza nuevos límites, deja que los dependientes inyectados y los límites de Promise retornada se drenen, y luego desactiva el `AsyncLocalStorage` subyacente; el trabajo no retornado permanece en propiedad del subsistema que lo desacopló. Si la cadena asíncrona heredada de un límite inicia la descarga de un fiber Cordis propietario, esa cadena de límite anidada se libera del drenaje para que la descarga no pueda esperarse a sí misma; sus continuaciones observan el servicio ya eliminado después del teardown. La [decisión del ámbito del iniciador](../../../.agents/notes/implemented/architecture/2026-07-15-agent-initiator-scope.md) es la dueña del contrato detallado de límites y teardown.

#### Factory API (creación)

La *creación* de agents la proporciona el plugin que implementa `AgentFactory` (`dsh-agent-loop`), registrado mediante `setFactory`. Esto mantiene la creación en la interfaz de `dsh-agent` para que los consumidores (la UI, el bridge ACP) programen contra `ctx.agents` sin depender del paquete concreto del loop. El registro canonicaliza un Service ya trazado a su destino concreto y vuelve a trazar cada llamada a través del contexto del llamador; esto evita las sombras Cordis anidadas a la vez que pasa un `ownerCtx` explícito vinculado al llamador a las fábricas simples.

- `ctx.agents.setFactory(factory: AgentFactory): () => void` — registra la fábrica de creación (el loop la llama en la construcción). Lanza una excepción ante una segunda fábrica; el slot se limpia al eliminar.
- `ctx.agents.create(options: CreateAgentOptions): Promise<AgentHandle>` — crea una sesión y un agent, espera el setup opcional mientras no se ha publicado, y luego publica a través de las comprobaciones finales de `SessionStore.enter()` y `AgentRegistry.enter()`. La creación concurrente con el mismo ID no se admite: más de una operación puede prepararse, pero solo una puede entrar; todo perdedor revierte su scope/sesión/driver privados. Un `signal` opcional, solo de creación, cancela el setup sin publicar y se desacopla antes de devolver el handle; la cancelación posterior usa `handle.dispose()` o `agent.cancel()`. La publicación está cubierta por rollback y cada borde de creación entregado se empareja durante el rollback. Rechaza si no hay ninguna fábrica registrada.
- `ctx.agents.resume(options: ResumeAgentOptions): Promise<AgentHandle>` — carga una sesión persistida ([persistencia de sesión](../../../.agents/notes/implemented/architecture/2026-06-14-session-persistence.md)), acuña un ámbito de agent nuevo sin publicar, espera el setup opcional y usa la misma secuencia de publicación de entrada final. Su `signal` opcional es asimismo solo de creación. Rechaza si no hay ninguna fábrica registrada o la persistencia de sesión no está configurada.

`AgentHandle = { agent: Agent; dispose(): Promise<void> }`. El disposer es una **capacidad de Consumer** — ningún observador que tenga la mera entrada del registro puede desmontar al agent. El fiber llamador y el provider de la fábrica registrada son copropietarios estructurales: la descarga del llamador impone la propiedad estructurada, mientras que la descarga de la fábrica debe detener las instancias antiguas porque su surface de dependencias con ámbito pertenece a ese provider. `dispose()` desde cualquier propietario alcanza un único límite de quiescence memoizado: detiene el loop, espera su salida, desregistra al agent, elimina su sesión del store y, por último, desenrolla su mundo con ámbito. `ctx.agents.get(id)` sigue devolviendo un `Agent` escueto; el bridge ACP y los backends de subagentes en proceso mantienen handles de consumer, mientras que los agents creados por configuración ya son propiedad del fiber del loop.

### Eventos en vivo

`dsh-agent` declara el vocabulario de coordinación en vivo `agent/*` para que los plugins no dependan del loop concreto. Las firmas exactas, los modos de dispatch, las reglas de filtrado por ámbito y los contratos de payload viven en la región generada de [core.md](../../../docs/subsystems/core.es.md#cordis-surface); el [flujo de turnos de la arquitectura](../../../docs/architecture.es.md#turn-flow) muestra su orden relativo a los eventos de sesión duraderos.

Los bordes del ciclo de vida tienen dos salvedades locales importantes. `agent/created` se ejecuta después del setup con ámbito y después de que existan tanto la entrada de la sesión como la del registro de agents. El setup es código de confianza, solo de composición; la notificación inmediatamente posterior, sin veto, `agent/session-start`, es el primer punto de inyección de arranque admitido. `agent/disposed` siempre significa que el agent exacto ha salido del registro. AgentLoop lo emite después de que su driver esté en quiescence, mientras que el teardown ordenado aún puede estar desacoplando la sesión y desenrollando el ámbito; los agents personalizados registrados directamente son dueños por sí mismos de cualquier contrato más fuerte de ordenación del driver.

La mayoría de los puntos de intercepción son waterfalls (cascadas de eventos) cooperativos. `agent/pre-step` recibe un payload que transporta el sujeto `agent`, el `UserMessage[]` reclamado en exclusiva, y el `turn`, `step` y `signal` de cancelación propuestos; su lote puede estar vacío cuando las herramientas ya requieren otra solicitud. Los puntos de extensión de turno con ámbito de agent llevan su `AbortSignal` explícito en el payload; los puntos de extensión restantes con ámbito de turno lo reciben a través de su valor de solicitud. Los listeners pueden cooperar con un signal, pero no deben retenerlo como autoridad sobre otro turno. `agent/request-error` es el waterfall de recuperación de las solicitudes de modelo fallidas: recibe las coordenadas de la solicitud, los hechos de fallo normalizados, la política de reintento de la inscripción que lo sirve cuando está disponible, y el signal. Un listener devuelve `{ kind: 'retry' }` sin llamar a `next()` cuando es dueño de la recuperación. `agent/turn-stopping` se ejecuta antes de que se cierre un turno por lo demás completado. La [decisión de cancelación explícita](../../../.agents/notes/implemented/architecture/2026-07-16-explicit-turn-cancellation.md) es la dueña de la vida del signal; la [Agent Note de diseño del runtime del agent scope](../../../.agents/notes/implemented/architecture/2026-07-12-agent-scope-runtime-design.md#three-execution-boundaries-are-deliberately-one-way) es la dueña del dispatch con ámbito y del cierre terminal.

`PreStepDecision` es o bien `{ kind: 'reject' }` o bien `{ kind: 'enter', messages }`. La rama enter es el lote completo, identificado y congelado para el step propuesto. Un listener que envuelve la entrada descendente preserva ese lote salvo que lo reemplace a propósito; las adiciones siguen el orden natural de retorno del waterfall. El reclamo ya eliminó del inbox los mensajes ofrecidos, por lo que el rechazo no los retiene. Los mensajes insertados después del reclamo permanecen pendientes para un límite posterior.

Las notificaciones en vivo del inbox son deliberadamente por mensaje y mínimas: `agent/inbox/inserted { message }`, `agent/inbox/claimed { message, turn }` y `agent/inbox/discarded { message }`. Complementan la proyección duradera `agent/inbox/spliced` sin añadir otro envoltorio de ciclo de vida.

Los límites de turno y de step y el flujo de tokens del modelo son hechos duraderos de `session/event` en lugar de notificaciones reflejadas de `agent/*`. Los Consumers leen `turn/*`, `step/*` y `assistant/chunk` del feed de la sesión; la política de herramientas y la observación de resultados pertenecen al pipeline completo documentado por [`dsh-tools`](../tools/README.es.md).

`foldConsumedWork(events)` vuelve a leer ese feed para responder a la única pregunta que la secuencia de turnos no puede responder por sí sola: qué fue del trabajo que consumió un log. Devuelve el `turn/end` más reciente que da cuenta del trabajo consumido — un turno que entró en un step de modelo, o uno que reclamó entrada del inbox y luego falló, fue detenido o fue rechazado antes de llegar a uno — además de si el trabajo aceptado fue cancelado después del inbox sin ejecutarse. Ambos hechos provienen del log, de modo que una cancelación se lee igual sea cual sea el propietario que la emitió. Un turno sin steps que no tomó nada, o que vació su reclamo y se completó, no describe trabajo alguno y se omite; un final `blocked` sobre entrada reclamada sí es una cuenta, porque el rechazo descartó esa entrada.

### Interfaz de agent (`types.ts`)

El handle contra el que programa cada plugin:

- `agent.inbox` — la proyección propiedad del agent de los eventos duraderos `agent/inbox/spliced`. `nextTurn` y `nextStep` exponen los valores `UserMessage` pendientes. `append`, `prepend`, `replace`, `remove`, `clear`, `splice` y `claim` los mutan; `replace(messageId, newMessage)` y `remove(messageId)` localizan el mensaje pendiente en ambas listas. El reemplazo puede cambiar la identidad y publica el mensaje antiguo como descartado seguido del mensaje nuevo como insertado. Las eliminaciones ordinarias y `clear()` son cancelaciones duraderas y emiten `agent/inbox/discarded`. `claim(target)` elimina el siguiente lote propuesto con splices de borrado puro; el loop emite entonces `agent/inbox/claimed`. `MessageId` es la única identidad de ocurrencia y debe permanecer única mientras esté pendiente.
- `agent.followup(message)` — pone en cola un mensaje ordinario `next-turn` y despierta al driver. No devuelve ningún handle de finalización; el id del mensaje identifica los hechos de inserción, reclamo y descarte del inbox, no una salida posterior ni un `turn/end`.
- `agent.steer(message)` — pone en cola entrada `next-step` que despierta. Un agent inactivo inicia un turno de forma síncrona; un driver en ejecución consume el steering (direccionamiento) posterior en su siguiente límite de step.
- `agent.inject(message)` — pone en cola contexto `next-step` que no despierta. Un driver en ejecución lo reclama en el límite pre-step posterior más cercano; un driver inactivo lo deja pendiente hasta que `followup()` o `steer()` despierten al driver. Puede perderse una solicitud cuyo pre-step ya reclamó su lote.
- `agent.cancel(cause, options?)` — cancela al driver activo y, salvo que esté presente `options.keepInbox`, cancela de forma duradera todo el trabajo pendiente del inbox. La cancelación en inactivo es un no-op.
- `agent.whenIdle()` — observa la quiescence de todo el agent, incluido el trabajo de reemplazo programado antes de que el driver actual se retire. No resuelve ningún mensaje en particular.
- `agent.session`, `agent.status`, `agent.options`, `agent.id`, `agent.ctx`

`running` describe un intervalo de drenaje de todo el driver, no una prueba de que un turno siga abierto; puede cubrir el cierre del turno, el punto de control de durabilidad y los turnos consecutivos en cola. Solo un llamador que sea dueño de un intervalo completo puede resumirlo como resultado de ejecución ([decisión](../../../.agents/notes/implemented/architecture/2026-07-30-followup-enqueue-and-owned-runs.md)).

### Puntos de extensión

- Creación de agents: `AgentLoop.create()` es la implementación concreta de la ruta de configuración (en `dsh-agent-loop`), mientras que los consumidores programáticos crean/reanudan agents propios mediante `ctx.agents.create()` / `ctx.agents.resume()`. Reemplaza el loop implementando `Agent` y registrándolo a través de `ctx.agents.register()`.
- Listeners de eventos: todos los eventos `agent/*` se declaran aquí — sin dependencia del paquete del loop.
- La delegación de subagentes (subagent) no es un método de `Agent`; los providers crean o conducen handles ordinarios a través de la API de fábrica, de modo que los transportes de delegación permanecen fuera de la interfaz central de agent.

## Experiencia de modelo

### Mensajes de usuario, steering e inyectados

#### Lo que ve el modelo

`send`, `steer` e `inject` alimentan a la sesión propietaria. `agent/pre-step` y otros eventos declarados permiten a los plugins rechazar un step propuesto o añadir material de solicitud duradero; esta interfaz no aporta prosa fija por sí misma.

#### Efecto en tokens

El contenido aceptado pasa a ser historial retenido o un prefijo de sesión repetido; el contenido bloqueado no aporta tokens de solicitud. El tamaño depende del llamador y del plugin.

#### Efecto en la caché KV

El historial y el steering aceptados son de solo añadido; un envío bloqueado no manda ninguna solicitud. Un prefijo de sesión permanece estable dentro de su instancia de loop, mientras que una instancia nueva o reanudada puede establecer un prefijo distinto.

### Composición de solicitud con ámbito de agent

#### Lo que ve el modelo

Los registros a través de `agent.ctx` pueden ensombrecer secciones de prompt o herramientas y pueden instalar interceptores solo para el agent durante el setup sin publicar.

#### Efecto en tokens

El paquete no añade tokens por sí mismo; las contribuciones con ámbito afectan solo a ese agent y desaparecen en la eliminación.

#### Efecto en la caché KV

Estable en el prefijo mientras las registraciones con ámbito de un agent no cambien. Un setup o una recarga que cambie secciones de prompt, definiciones de herramientas o listeners de solicitud puede invalidar la reutilización desde el primer token de solicitud afectado.

## Limitaciones conocidas y trabajo diferido

- **El ámbito del iniciador es local al proceso** — los workers, los procesos hijos, HTTP, las colas duraderas y los reinicios materializan explícitamente cualquier identidad requerida.
- **La identidad ambiental puede sobrevivir a la vida** — los consumidores comprueban igualmente `agent.status`, la cancelación y el contrato de capacidad propietaria antes del trabajo sensible al ciclo de vida.
- **Canales entre agents más allá de la delegación** — el estado compartido, la salida de los hijos en streaming y las semánticas de fondo/poll permanecen fuera del seam síncrono actual `ctx.subagents`.
- **`agent/session-start` no puede condicionar el arranque** — sigue siendo una notificación síncrona sin veto; la composición asíncrona que deba terminar antes de la publicación pertenece a la transacción `setup(agentCtx)` de la fábrica.
- **`cancel()` vacía el inbox por defecto** — aborta el turno en curso además del trabajo en cola y de steering; `cancel(cause, { keepInbox: true })` aborta solo el turno y preserva los elementos pendientes. Sigue sin existir un abort solo de step que mantenga en marcha el turno en curso ([Agent Note de la stop API](../../../.agents/notes/implemented/simplification/2026-06-20-public-agent-stop-api.md)).
- **Cada `UserMessage` adicional transporta exactamente un `MessageSource`** — las contribuciones de varios plugins fusionadas en una sola llamada de herramienta colapsan bajo un único source, de modo que el mensaje no puede nombrar a varios productores.
- **`SessionStartSource` reserva `'clear'`/`'compact'` sin emisor todavía** — solo ocurren `'startup'`/`'resume'` hasta que se materialicen los subsistemas impulsores (`TODO(compaction)`).
