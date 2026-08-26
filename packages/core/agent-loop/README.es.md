# dsh-agent-loop

[English](README.md) | Español

EL plugin de agent concreto y el driver del loop. Su implementación interna al paquete satisface la interfaz `Agent` y conduce el ciclo de vida de sesión/turno/paso.

Es el único paquete del harness que contiene lógica de loop concreta. Todo lo demás es un servicio abstracto o un plugin contra puntos de extensión — el comportamiento nuevo va a los plugins, no aquí.

## Servicio: `AgentLoop` (clave de ctx: `agentLoop`)

### API pública

La creación y la reanudación son una sola transacción cubierta por rollback: se construye una sesión privada, un agent concreto y un contexto con ámbito; se espera el setup opcional; se entra en ambos registros; se anuncia `session/created` y luego `agent/created`; se emite `agent/session-start`; y solo entonces se arranca el driver. El setup recibe el `Context` con ámbito completo como código de composición de confianza dentro del mismo proceso y no debe conducir al agent no publicado. Las entradas ordinarias de identidad tipada y de opciones se toman prestadas bajo su contrato de solo lectura, mientras que los eventos seed y los metadatos de sesión se validan y se toman instantáneas de ellos porque cruzan la frontera de la sesión duradera. Un `AbortSignal` opcional cancela solo la carga/setup/publicación y se desacopla antes de que el identificador devuelto sea visible.

La fibra llamante y el provider de AgentLoop son co-propietarios. `AgentFactory.createAgent(ownerCtx, options)` y `resume(ownerCtx, options)` reciben la propiedad del llamante de forma explícita, mientras que la fábrica conserva su propio contexto de dependencias para `sessions`/`llm`/`tools`/`systemPrompt`; esto permite que un llamante inyecte solo `agents` sin encoger el conjunto de servicios del nuevo agent. La descarga del llamante, la disposición del identificador o la descarga del provider convergen en un único límite de quiescencia memoizado. El apagado del provider espera tanto la limpieza de recursos como el wrapper público de create/resume que observó la desactivación, de modo que ninguna continuación pueda publicar después de que desaparezcan las dependencias.

Cada agent y su sesión comparten un `SessionId` elegido por el llamante, que se asume globalmente único; las colisiones accidentales de UUID quedan fuera del modelo soportado. Dos operaciones concurrentes con el mismo id pueden prepararse ambas, pero las llamadas finales a `enter()` arbitran la publicación y todo perdedor revierte sus recursos privados. Cada detach está ligado al objeto exacto en el que se entró, así que un disposer obsoleto no puede eliminar un reemplazo posterior con el mismo id. Un detach solicitado durante una notificación de creación síncrona espera a que ese despacho se desenrolle, preservando el emparejamiento created/disposed. El teardown ejecuta stop y drain → desenrollar el ámbito → desacoplar el agent → desacoplar la sesión; el id se vuelve reutilizable después de la limpieza del ámbito privado. Las notificaciones ordinarias no vetantes de `agent/*` pasan por `agentEvents(ctx, agent)`, y el ensamblaje por paso pasa por `assembleContextFor(agent)`.

- `ctx.agentLoop.create(id: SessionId, options?: AgentOptions, meta?: { cwd?: string }): Agent` — create síncrono sin setup bajo el id compartido exacto de agent/sesión, dispuesto con la fibra llamante. La config declarativa trata `agents[].id` como una etiqueta estable y normalmente acuña `${label}-session-<uuid>` antes de llamar a esta frontera. Una app puede en su lugar suministrar un `sessionId` exacto y estable: el primer uso lo crea, mientras que un remontaje con persistencia ya presente reanuda su historial materializado. `resumeSessionId` exige y carga un id persistido existente y es mutuamente excluyente con `sessionId`. Esto mantiene los reinicios frescos por defecto sin colisiones y sin retener una segunda identidad de enrutamiento viva.

`AgentLoop` también implementa el contrato `AgentFactory` y se registra a sí mismo mediante `ctx.agents.setFactory(this)`, así que los plugins crean/reanudan agents a través de `ctx.agents`:

- `ctx.agents.create({ sessionId, meta?, seed?, agentOptions?, setup?, signal? }): Promise<AgentHandle>` — create programático bajo el id compartido suministrado por el llamante. Espera la transacción de setup no publicada antes de devolver; `meta` lleva metadatos de cwd/linaje/frontera de seed y `seed` reconstruye un prefijo hijo con fork después de que la frontera de sesión valide y tome instantánea de los valores duraderos. `signal` solo se aplica hasta que esta promesa se resuelve. El [`AgentHandle`](../agent/README.es.md) resuelto es dueño del teardown exacto.
- `ctx.agents.resume({ resumeSessionId, agentOptions?, setup?, signal? }): Promise<AgentHandle>` — carga una sesión persistida mediante `ctx.sessionPersistence` ([persistencia de sesión](../../../.agents/notes/implemented/architecture/2026-06-14-session-persistence.md)), registra el agent bajo ese mismo id, reconstruye su historial y luego espera el setup contra un ámbito de agent no publicado y fresco antes de la publicación cubierta por rollback. La numeración de turnos y el historial derivado continúan desde el registro cargado. Exige un backend de persistencia de sesión (NO inyectado en duro — las demos no persistentes siguen funcionando; `resume` rechaza con un error claro cuando la persistencia está ausente). `signal` es solo de creación. Devuelve un `AgentHandle`.

El camino `ctx.agentLoop.create()` dirigido por config mantiene su agent en propiedad de la fibra del loop (descarta el identificador). Para un agent programático, el portador del identificador es la única capacidad de teardown orientada al consumidor; la descarga del provider de AgentLoop es el borde estructural de teardown independiente, no otro identificador expuesto al código de aplicación.

### Servicios inyectados

`agents`, `sessions`, `llm`, `tools`, `systemPrompt` — los cinco servicios de interfaz.

### Compañero de invariante

El compañero opcional `@deepseek-ai/dsh-agent-loop/invariant` registra la reconstrucción de solicitudes con `ctx.invariants`. El loop registra cada solicitud congelada exacta en el conjunto de identidad local al proceso propiedad de `dsh-llm`; el compañero exige entonces una sesión viva y reconstruye de forma independiente la frontera de mensajes y la cabecera de solicitud plegada desde el registro. Las llamadas directas de un solo disparo quedan fuera de este contrato incluso cuando los llamantes las congelan o les adjuntan un id de sesión.

### Configuración (schemastery)

```ts
interface Config {
  maxParallelToolCalls?: number // default 10; 1 is serial
  agents: Array<{
    id: string                 // required
    provider?: string
    model?: string
    maxTokens?: number         // positive per-request output-token cap
    resumeSessionId?: string   // load this persisted session instead of creating one
    cwd?: string               // optional workspace cwd for the fresh session
  }>
}
```

Los agents configurados arrancan automáticamente. Una llamada al modelo exige `provider` y `model`; `agent/request` puede suministrar el par faltante antes del despacho. Un `maxTokens` positivo opcional siembra el tope de salida de cada solicitud de conversación y se registra en su cabecera de solicitud. `maxParallelToolCalls` limita el pool rodante de cada agent para llamadas seguras en paralelo y su valor por defecto es `10`; es también toda la sección de ajustes de `agent-loop`, así que una capa de usuario sobre esta entrada limita el siguiente grupo de herramientas sin reinicio, y un valor que no sea un entero positivo se rechaza en la escritura, no en ese grupo. `agents` está deliberadamente ausente de esa sección — se consume una sola vez cuando el servicio arranca, así que un cambio almacenado solo podría parecer que tuvo efecto. `cwd` solo se aplica a sesiones nuevas, mientras que `resumeSessionId` retiene los metadatos persistidos. Los agents configurados usan la persona del despliegue, y el setup programático puede sombrearla por agent. Este plugin suministra las variables de prompt `provider`, `model` y `cwd` por agent; la identidad del harness y la persona del despliegue pertenecen a `dsh-system-prompt`.

### Driver concreto interno

El `ReactLoopAgent` concreto, su bandeja de entrada y sus controles de ejecución son internos al paquete. La raíz del paquete exporta solo el contrato de plugin/servicio/config, y el mapa de exports del paquete no expone ninguna vía de escape `./src/*`; los propietarios del ciclo de vida crean agents a través de `ctx.agents` en lugar de nombrar, construir o arrancar internos del driver. Una sesión preparada solo puede ser reclamada por un driver concreto, y todo lo observable sucede a través de los eventos de sesión y la taxonomía de eventos `agent/*`.

La primitiva unificada `send()` enruta contenido y fuente por (`target` × `wakeup`); `followup`/`steer`/`inject` son sus alias de preset fijo. `followup()` añade a la FIFO `next-turn` y despierta al driver, `steer()` añade a la bandeja de entrada `next-step` y la despierta, e `inject()` añade a esa misma bandeja de entrada `next-step` sin despertarla. En una frontera de turno el driver abre el turno duradero y luego reclama atómicamente la entrada de `next-step` pendiente más un prompt encolado; entre pasos solo reclama entrada de `next-step`. Reclamar retira el lote mediante empalmes de borrado puro y emite `agent/inbox/claimed { message, turn }` una vez por mensaje. `agent/pre-step` devuelve entonces o un rechazo o los mensajes completos que entran en el paso propuesto. El rechazo deja el lote reclamado retirado y cierra el turno sin paso; la entrada insertada después de la reclamación permanece pendiente, y la inyección ociosa espera hasta que un follow-up o el steering despierte al driver.

Cada mutación de la bandeja de entrada publica un evento normalizado `agent/inbox/spliced` antes de cambiar la proyección viva. Las inserciones, ediciones, retiradas, reclamaciones y cancelaciones se reproducen a través de las mismas coordenadas de empalme estándar. Las retiradas ordinarias llevan `outcome: 'canceled'` y emiten `agent/inbox/discarded { message }`; reclamar usa borrados puros sin resultado, tras lo cual el loop emite `agent/inbox/claimed`. Cada inserción emite `agent/inbox/inserted { message }`. `MessageId` sigue siendo único en ambas listas pendientes, y los observadores síncronos de eventos duraderos pueden reconstruir los valores retirados desde la proyección previa al empalme.

### Ciclo de vida del loop (`agent.ts`)

El driver es dueño de un agent durante toda su vida y se ejecuta dentro de `ctx.agents.withInitiator(agent, ...)`. Los puntos de entrada de orquestación privados al paquete recuperan el Agent exacto, derivan `agent.session` una sola vez y dejan que los helpers locales a la operación lo capturen en lugar de reenviar el driver concreto o la `Session` por operación a través de interfaces superficiales. Un helper conserva una `Session` explícita cuando esa es su interfaz real, mientras que la creación, la carga de persistencia, el setup no publicado, los servicios, los workers, los procesos, la persistencia y los protocolos wire retienen sus identidades explícitas. El [servicio agent](../agent/README.es.md#initiating-agent-scope) es dueño de las reglas de propagación, teardown y trabajo desacoplado.

Cada llamada de provider que alcanza una finalización exitosa añade exactamente un ancla de completado `assistant/message`, incluidas las llamadas sin contenido y las finalizaciones por max-tokens. El ancla registra el contenido ensamblado tal cual, lista las secuencias exactas de chunks en `sourceEventSeqs` (`[]` para un stream sin chunks) e incluye el uso cuando está disponible; el contenido vacío queda fuera del historial de mensajes derivado. Una cancelación de turno que interrumpe el streaming también añade un ancla con `interrupted: true` cuando texto o razonamiento no vacío ha llegado al usuario. El ancla cita esas secuencias de chunks y coloca el prefijo renderizado en el historial de mensajes derivado, así que la siguiente solicitud contiene lo que el usuario vio. Las llamadas de herramienta no despachadas se omiten, y un stream vacío o de solo herramientas no produce ancla; los fallos de provider tampoco confirman contenido de assistant ([decisión](../../../.agents/notes/implemented/architecture/2026-08-10-cancelled-stream-prefix-finalize.md)).

Después de que `agent/request` devuelve una config de llamada de provider/modelo, el loop pide a `ctx.llm.prepareCall()` que valide los campos propiedad del adaptador y materialice los valores por defecto configurados de reasoning-effort y de tope de tokens de salida bajo la señal del turno activo. La llamada preparada retiene el registro exacto del adaptador a través de esta resolución asíncrona, del registro `request/header` y del despacho terminal, así que el HMR no puede mezclar el resultado de capacidad de un adaptador con la solicitud de otro. La cabecera registra la config efectiva y qué campos vinieron del adaptador. Antes del siguiente waterfall, el loop retira esos campos marcados de la propuesta para que la ruta exacta vigente rematerialice sus propios valores por defecto; los ajustes explícitos no marcados persisten a través de los pasos y de los cambios de ruta. Una ruta sin adaptador registrado conserva la config propuesta para que un listener de `llm/stream` pueda poseerla y cortocircuitarla; el despacho terminal no manejado sigue fallando con `NO_ADAPTER`. Una nueva instancia del loop sigue la misma regla de marcado de valores por defecto del adaptador al reanudar.

El fallo de un plugin termina el turno actual, no el loop. Los fallos finales de selección de adaptador, despacho e iteración llegan desde `ctx.llm` como finalizaciones de error terminal o abortado y entran en `agent/request-error`; los fallos de middleware, de procesamiento de resultados, de herramientas y de otras extensiones permanecen lanzados y cierran directamente. La recuperación recibe coordenadas de solicitud, hechos inmutables de provider, la política de reintento inmutable capturada por el registro del adaptador preparado y la señal de turno; la política está ausente cuando el middleware es dueño de una ruta no preparada. Un listener que la maneja devuelve `{ kind: 'retry' }`; un fallo no manejado es terminal. AgentLoop es dueño de una señal de cancelación para la admisión o el turno actuales. Un `cancel(cause)` efectivo limpia el trabajo pendiente a menos que `keepInbox` esté fijado y aborta cooperativamente esa señal; la cancelación ociosa es un no-op. La entrada que despierta y llega después del aborto pero antes de que la actividad converja al estado ocioso queda retenida (`wakeRequested`) y se reproduce en el propio límite de convergencia del driver, así que se ejecuta sin un envío de despertado adicional; una cancelación `disposed` nunca retiene, y un despertado enviado mientras ya está ocioso siempre abre su frontera de turno (el estado muestra un par transitorio `idle → running → idle` incluso cuando el mensaje se limpió). El `turn/end` duradero registra `aborted` para `user` y `parent`, mientras que la disposición registra `disposed`; las llamadas de herramienta de modelo no despachadas reciben pares sintéticos `tool/call` y de resultado `ABORTED_BEFORE_DISPATCH`. La causa de cancelación cambia la notificación, no cómo se finaliza el contexto de resultados tras la cancelación. La disposición espera al trabajo que ignora la señal antes de la retirada del registro. La [decisión de cancelación explícita](../../../.agents/notes/implemented/architecture/2026-07-16-explicit-turn-cancellation.md) y el [pestillo de despertado de convergencia de cancelación](../../../.agents/notes/implemented/bug-fix/2026-08-07-cancel-convergence-wake-latch.md) son dueños del contrato de ciclo de vida y de carrera.

Dentro de un paso, las llamadas exclusivas forman barreras; las llamadas seguras en paralelo usan un pool rodante acotado y se reclasifican antes de empezar. Solo se solapan el despacho y el cuerpo. La política, los resultados duraderos y el contexto de resultados permanecen ordenados por el modelo. El aborto detiene nuevas llamadas, drena los resultados ya iniciados y retiene su contexto de resultados finalizado sin distinguir la causa de cancelación. Un fallo interno del planificador detiene nuevos despachos, espera a los despachos ya iniciados y alcanza la frontera de error de turno sin fabricar resultados de herramientas.

### Qué pertenece a los plugins

Todo lo que va más allá de «llama al modelo, ejecuta las herramientas, repite» pertenece a los plugins que escuchan en la taxonomía de eventos:
- Hooks y política: los checkpoints relevantes de `agent/*` más el pipeline con guards `tools/pre-execute` → `tools/execute` → `tools/post-execute` → `finalizeContent` propiedad de la definición → `tools/result`; las firmas y los modos exactos de los eventos están en las regiones generadas de [core.md](../../../docs/subsystems/core.es.md#cordis-surface) y [tools.md](../../../docs/subsystems/tools.es.md#cordis-surface)
- Compactación: presión en `agent/pre-step`; reparación canónica de desbordamiento en `agent/request-error`
- Recuperación de solicitudes al modelo: `dsh-llm-retry` registra y espera backoff normal o ilimitado del provider exacto en `agent/request-error`, emite el estado no superficial de `llm/retry` y devuelve entonces una acción de reintento
- Sandbox, permiso, modo plan: `tools/pre-execute` para deny/ask extensible, `tools.guard()` para política de propietario monótona, `tools/post-execute` para decisiones de resultados y `tools/result` para la observación final
- Subagentes: implementados fuera del loop como providers de `ctx.subagents`; los providers en proceso usan `ctx.agents.create()` y el teardown del `AgentHandle` que ellos poseen, mientras que el genérico [`ctx.jobs`](../../jobs/jobs/) más [`dsh-tool-subagent`](../../subagent/tool-subagent/) son dueños de la recolección en segundo plano.
- Persistencia: escritura diferida inmediata desde `session/event`; `session/flush` es una barrera de observación explícita
- UI: `session/event` (stream de tokens de assistant, fronteras, actividad de herramientas) + eventos de control `agent/*` (`agent/status`, `agent/created`/`agent/disposed`)

## Experiencia del modelo

### Solicitud de conversación completa

#### Qué ve el modelo

Para cada paso, el loop envía el system prompt por agent renderizado, los schemas de herramientas visibles y los mensajes derivados de la sesión. Suministra los valores de las variables `provider`, `model` y `cwd` pero ninguna prosa fija adicional.

#### Efecto de tokens

El texto del sistema y los schemas se pagan de nuevo en cada paso. El ámbito por agent elige las contribuciones, mientras que el waterfall de ensamblaje autoritativo puede alterar la solicitud final y hace a su listener responsable de la coherencia del protocolo.

#### Efecto de KV Cache

De solo anexión solo mientras el texto del sistema, los schemas y el historial anterior permanezcan byte a byte idénticos bajo la misma ruta de provider y modelo. Una reescritura del ensamblaje con tokens o un cambio de composición pueden invalidar la reutilización desde el primer token de solicitud alterado.

### Historial de mensajes retenido

#### Qué ve el modelo

Los mensajes de usuario aceptados, los mensajes de assistant, las llamadas y los resultados de herramientas, el contexto inyectado y el steering se registran y se envían en pasos posteriores. Los chunks crudos de stream, las fronteras de ciclo de vida y otros eventos de solo registro se excluyen.

#### Efecto de tokens

La entrada crece con cada mensaje superficial hasta que un reemplazo de compactación sombrea los nodos más antiguos; un turno de herramientas de varios pasos reenvía el historial acumulado en cada paso.

#### Efecto de KV Cache

El crecimiento ordinario del historial es de solo anexión y preserva las entradas reutilizables. Un reemplazo superficial o una compactación invalidan la reutilización desde el primer token de historial sombreado.

### Llamadas no despachadas tras la cancelación

#### Qué ve el modelo

Si una solicitud posterior reproduce un paso abortado, cada llamada de herramienta cuya cancelación impidió el despacho tiene el código de error `ABORTED_BEFORE_DISPATCH` y el texto de resultado `Error: tool call aborted before dispatch`.

#### Efecto de tokens

Un resultado de error fijo por llamada omitida permanece en el historial hasta que la compactación lo sombrea.

#### Efecto de KV Cache

De solo anexión; cada resultado sintético sigue el prefijo de solicitud reutilizable y no invalida las entradas de KV Cache existentes.

## Limitaciones conocidas y trabajo diferido

- **La clasificación es unaria** — las llamadas cuya seguridad depende de comparar hermanos o recursos deben permanecer exclusivas ([fundamento](../../../.agents/notes/implemented/feature/2026-07-10-parallel-tool-call-execution.md)).
- **Las etiquetas de config son nuevas por defecto** — omitir `sessionId` crea un `${id}-session-<uuid>` nuevo en cada arranque; el comportamiento exacto de reanudar-o-crear exige un `sessionId` estable explícito, mientras que `resumeSessionId` exige historial persistido existente.
- **Los agents de config no tienen campo de persona por agent ni hook de setup** — usan la persona del despliegue; la composición de persona/herramientas con ámbito solo está disponible a través de las opciones de fábrica programáticas `ctx.agents.create()` / `resume()`.
- **Sin presupuesto de turno incorporado** — las llamadas de herramientas o el steering continúan el turno actual; una política que acote los turnos desbocados debe cancelar desde un punto de extensión de ciclo de vida existente como `agent/turn-stopping`.
