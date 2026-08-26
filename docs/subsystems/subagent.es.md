# Subagent

[English](subagent.md) | Español

El seam de subagente (subagent) permite que un agent (agente) delegue trabajo a un agent hijo. Como [bash](shell.es.md), es **una capacidad opcional**, no parte del agent loop (bucle del agente), por lo que sus tipos viven aquí y no en [core.md](core.es.md). Se diferencia de los demás seams de capacidad porque **coexisten múltiples implementaciones de provider** en un mismo contexto, registradas por nombre (`ctx.subagents`), mientras que bash solo admite un ejecutor. Su registro sigue el [registro de adaptadores LLM (modelo de lenguaje de gran tamaño)](llm-streaming.es.md), no el ejecutor bash de servicio único.

Service Definition: [dsh-subagent](../../packages/subagent/subagent) (`ctx.subagents` + el vocabulario de abajo). Los Service Providers son paquetes hermanos (`dsh-subagent-spawn-in-process`, `-fork`, `-acp`, `-codex`, `-claude-code`, `-dsh-sdk`); los Consumers orientados al modelo son [dsh-tool-subagent](../../packages/subagent/tool-subagent) (delegación por provider), [dsh-tool-subagent-control](../../packages/subagent/tool-subagent-control) (los controles globales opcionales `send_message`, `interrupt_agent` y `list_agents`) y [dsh-tool-subagent-report](../../packages/subagent/tool-subagent-report) (el canal de retorno `report` opcional con ámbito de hijo). El mismo servicio `ctx.subagents` es dueño de la orquestación de hijos continuables a través de un gestor de activación interno y del descubrimiento de solo lectura de hijos y descendientes directamente desde el almacén de sesiones y la persistencia de sesiones opcional. La justificación de los providers de producto vive en [el Agent Note de Codex y Claude Code](../../.agents/notes/implemented/feature/2026-08-04-claude-code-and-codex-subagent-backends.es.md); la justificación del seam común vive en [el Agent Note de subagent](../../.agents/notes/implemented/feature/2026-06-21-subagent-capability-seam.es.md), [el Agent Note de subagentes continuables](../../.agents/notes/implemented/feature/2026-07-28-continuable-subagent-conversations.es.md), [el Agent Note de la herramienta de report](../../.agents/notes/implemented/feature/2026-07-30-continuable-subagent-report-tool.es.md), [el Agent Note del catálogo durable](../../.agents/notes/implemented/feature/2026-07-22-durable-subagent-catalog-and-list-agents.es.md), [el Agent Note de la proyección de identidad de listado](../../.agents/notes/implemented/architecture/2026-08-06-subagent-list-identity-projection.es.md) y [el Agent Note del servicio fusionado](../../.agents/notes/implemented/simplification/2026-07-26-merge-subagent-control-service.es.md).

Fuentes: [`packages/subagent/subagent/src/types.ts`](../../packages/subagent/subagent/src/types.ts), [`packages/subagent/subagent/src/index.ts`](../../packages/subagent/subagent/src/index.ts) y [`packages/subagent/subagent/src/continuation.ts`](../../packages/subagent/subagent/src/continuation.ts)

## Dos tipos de capacidad, dos formas de descubrirlos

Un provider anuncia sus funcionalidades **de tiempo de arranque** en un descriptor estático que el servicio comprueba ANTES de que exista una ejecución one-shot; una solicitud que necesita una funcionalidad de la que el provider carece se rechaza ruidosamente (`SubagentError('UNSUPPORTED_CAPABILITY')`), nunca se acepta y luego se ignora. Esas banderas describen solo la ruta one-shot de [`start()`](#the-provider-contract-subagentprovider), en la que el provider compone al hijo. Los hijos **continuables** los compone el propio gestor de continuación, así que están controlados por un método opcional cuya presencia ES la capacidad, con el estrechamiento (narrowing) de TS como mecanismo de descubrimiento: [`SubagentProvider.prepareContinuable`](#the-provider-contract-subagentprovider).

```ts type-equiv
/**
 * Which START-TIME features a provider supports. Checked by the service before delegating to
 * {@link SubagentProvider.start}: a request that needs a capability the chosen provider lacks
 * is rejected with a typed error rather than accepted-then-ignored (the "fail loud, no silent
 * degradation" rule). These flags describe the ONE-SHOT
 * {@link SubagentProvider.start} path, where the provider composes the child;
 * continuable children are composed by the continuation manager itself and are
 * gated by {@link SubagentProvider.prepareContinuable} instead. Each flag
 * corresponds one-to-one to a {@link SubagentStartRequest} option: `depthLimit`
 * to `maxDepth`; the other names match.
 */
interface SubagentCapabilities {
  readonly outputSchema: boolean
  readonly depthLimit: boolean
  readonly toolFilter: boolean
  readonly persona: boolean
}
```

## La solicitud de inicio one-shot

La capa de herramientas construye esta solicitud a partir de la entrada del modelo y de su propia configuración; el servicio la valida contra el provider nombrado antes de `start`. El `parent` obligatorio aporta el cwd de la sesión, el linaje y la profundidad de delegación. El schema de salida, la profundidad, el filtro de herramientas y la persona opcionales exigen banderas de capacidad correspondientes. Los schemas no soportados fallan en el arranque; los backends en proceso acotan filtros y personas a la creación del hijo e implementan el schema soportado con raíz en objeto mediante una herramienta de captura forzada.

```ts type-equiv
/**
 * What a caller asks for when starting a ONE-SHOT subagent. The tool layer
 * builds this from the model's `{ description, prompt }` plus its own config;
 * the service validates {@link SubagentCapabilities} against the named provider
 * and resolves the durable descriptor before dispatching to
 * {@link SubagentProvider.start}.
 */
interface SubagentStartRequest {
  /** Optional short display label persisted with a session-backed child. */
  readonly label?: string
  /** Content delivered as the child's user message. */
  readonly prompt: ContentBlock[]
  /**
   * The spawning agent. In-process providers derive workspace, lineage, and
   * delegation depth from its durable session state. ACP reads only its cwd,
   * and only when no deployment `cwd` override is configured.
   */
  readonly parent: Agent
  /**
   * Cancellation signal from the spawning context (the tool's `exec.signal`).
   * This is the canonical cancellation channel both before and after startup:
   * a provider rejects `start()` after cleaning partial resources when it
   * fires before the run is published, and cancels the published run's
   * remaining turn work when it fires afterward.
   */
  readonly signal: AbortSignal
  readonly agentOptions?: AgentOptions
  /**
   * Object-rooted JSON Schema within `assertObjectJsonSchema`'s enforced subset. Start rejects
   * unsupported schemas or providers without the capability. Data must be plain host-realm JSON;
   * a successful child returns the matching value as {@link SubagentResult.structured}.
   */
  readonly outputSchema?: ObjectJsonSchema
  /**
   * Optional absolute delegation-depth cap for the child being started: its
   * computed depth must be less than or equal to this non-negative safe
   * integer. Requires {@link SubagentCapabilities.depthLimit}; rejected at
   * start otherwise.
   */
  readonly maxDepth?: number
  /**
   * Optional child tool scoping. Requires {@link SubagentCapabilities.toolFilter};
   * rejected at start otherwise. In-process backends apply it as a scoped
   * `tools.restrict()` in the child's creation window: the named tools vanish
   * from the child's prompt AND refuse to execute (one visibility), with loud
   * unknown-name validation.
   */
  readonly toolFilter?: ToolRestriction
  /**
   * Optional per-child persona. Requires {@link SubagentCapabilities.persona};
   * rejected at start otherwise. In-process backends register it as a scoped
   * `deployment:persona` section on the child, SHADOWING the deployment's
   * persona for this child alone — same template semantics as the deployment
   * persona (strict `{{…}}` interpolation against the registered variables).
   */
  readonly persona?: string
}
```

`signal` es el único canal de cancelación antes y después de la preparación (readiness). La justificación de la persona, el filtro global de herramientas en vivo, la profundidad absoluta y la visibilidad-sin-autoridad vive en [el Agent Note de controles de composición de subagent](../../.agents/notes/implemented/feature/2026-07-12-subagent-persona-tool-filter-and-depth.es.md).

La solicitud orientada al llamador no transporta detalles del formato de catálogo ni estado de continuación. `SubagentRuntime.start()` resuelve el descriptor one-shot desprendido tras las comprobaciones de capacidad y luego pasa esta solicitud orientada al provider al transporte seleccionado; un hijo continuable nunca llega a `SubagentProvider.start()`:

```ts type-equiv
/**
 * Provider-facing one-shot request after {@link SubagentRuntime.start} resolves
 * the durable child descriptor.
 */
interface ResolvedSubagentStartRequest extends SubagentStartRequest {
  /** Detached descriptor a session-backed provider persists in the child log. */
  readonly descriptor: SubagentDescriptorData
}
```

## Hijos continuables y activaciones

Un **subagente continuable en segundo plano** es una Session durable de hijo con a lo sumo una **Activación** local al proceso: el período en el que un Agent hijo reconstruido está residente. Una Activación no es una solicitud, un resultado, una cancelación ni una tarea: puede ejecutar muchos turnos FIFO y permanece residente mientras los descendientes que creó sigan en ejecución. El gestor de continuación es dueño de la admisión de activaciones, la autorización del parent directo, el grafo de propiedad en vivo, la reanudación en frío y la liberación de hijos primero; el agent loop es dueño de todo el ordenamiento y la ejecución de turnos. Ninguna ruta continuable crea una tarea ni un envoltorio intermedio con resultado.

```text
persisted Session
  -> optional live Activation
       -> one retained AgentHandle
       -> Agent inbox as the only turn FIFO
       -> zero or more owned child Activations
```

`SubagentRuntime.startContinuable()` reserva el id estable del hijo, hace una instantánea del payload versionado `subagent/descriptor`, pide al provider nombrado su `ContinuableCreateSpec` desprendido, crea el Agent hijo a través de un ámbito privado de propietario de activación, establece la propiedad de parent continuable si procede y envía el prompt inicial. Se resuelve con `{ childId, messageId }` cuando la aceptación en el buzón produce el id del mensaje, sin esperar a que el turno arranque ni a que el mensaje entre en el registro de la Session. Todo fallo anterior a esa aceptación se rechaza sin ninguno de los dos ids, liberando cualquier handle creado y revirtiendo la Activación y la propiedad del parent.

`SubagentRuntime.followup()` es la única operación de mensaje de continuación, y el enrutamiento depende solo de la residencia de la Activación:

| Estado de la Activación | `followup` |
|---|---|
| `running` | encolar en la misma Activación |
| `waiting` | despertar la misma Activación |
| sin Activación | reanudar en frío una Activación nueva |

`running` significa que el Agent tiene una admisión o un turno activos, o trabajo de buzón despertando; `waiting` significa que está en quiescencia pero aún posee al menos una Activación de hijo que no ha completado su liberación; `settled` significa quiescencia con todos los hijos propiedad liberados, momento en el que el gestor libera el [`AgentHandle`](core.es.md#creation-and-ownership) y elimina la Activación. El gestor deriva estas condiciones internas de la quiescencia del Agent y del conjunto de hijos propiedad, en lugar de mantener una segunda máquina de estados de ejecución.

El buzón del Agent es la única cola. Cada mensaje de continuación se convierte en un turno FIFO de `Agent.followup()`, de modo que los mensajes aceptados tienen un único orden observable y un seguimiento no puede redirigir un turno ya en curso. La entrega correcta devuelve el `MessageId` aceptado; los eventos existentes `agent/inbox/inserted`, `agent/inbox/claimed` y `agent/inbox/discarded` siguen siendo las observaciones del ciclo de vida del mensaje, y la capa de continuación no define ninguna ruta de entrega específica de subagent.

La autoridad de seguimiento proviene de un contexto de herramienta exacto de un Agent vivo. El Agent autenticado debe ser el parent directo del hijo durable registrado en `SessionHeader.parentSession`. `MessageSource` y `senderSessionId` registran quién aportó un mensaje admitido, pero no conceden autoridad; la herramienta opcional orientada al modelo usa `CoordinatorMessageSource`.

En ambas operaciones, la señal del llamador es dueña de la búsqueda, la materialización y la admisión solo hasta la aceptación en el buzón. A partir de ahí, el gestor es dueño de la Activación de forma independiente: una cancelación posterior del llamador ni cancela el turno aceptado ni libera al hijo, y el seam no expone ninguna operación de steering (direccionamiento).

`SubagentRuntime.interrupt(targetSessionId, authority)` es la única parada pública: autoriza de forma síncrona, emite `Agent.cancel(cause, { keepInbox: true })` sobre el objetivo vivo y retorna sin esperar la quiescencia. La Activación, su trabajo pendiente no reclamado en el buzón y los descendientes publicados quedan intactos; el trabajo ya reclamado en el turno interrumpido no se reencola. Una vez que el driver interrumpido queda inactivo, un envío que lo despierta reanuda la cola FIFO aparcada. Un objetivo ausente —desconocido, one-shot o ya asentado— y una composición sin gestor son no-ops aceptados. Para un objetivo vivo, una dirección de parent no coincidente o un llamador fuera de su ascendencia viva se rechaza con `UNAUTHORIZED`; los objetos de ancestro obsoletos y las solicitudes de ancestro auto-referenciadas se rechazan antes de la búsqueda del objetivo.

```ts type-equiv
/**
 * Authority under which one interrupt request is admitted. `user` carries the
 * durable direct-parent address a human client presented; `ancestor` carries
 * the exact live Agent object whose recorded lineage must contain the caller.
 */
type SubagentInterruptAuthority =
  | { readonly kind: 'user'; readonly parentSessionId: SessionId }
  | { readonly kind: 'ancestor'; readonly agent: Agent }
```

Cada Activación posee su `AgentHandle` y un `ownedChildren: Set<SessionId>`; como una Session tiene a lo sumo una Activación viva, el id de la Session del hijo identifica al hijo vivo sin otra referencia de encarnación del runtime. Iniciar un hijo o enviar trabajo originado por el parent registra al hijo en el conjunto de un parent gestionado por la continuación antes de que el hijo pueda ejecutarse, y ese parent no puede asentarse mientras el conjunto no esté vacío. Un Agent de nivel superior u otro Agent no continuable no tiene Activación y permanece fuera del grafo de espera. La liberación del hijo ocurre solo después de que el Agent hijo esté en quiescencia, cada hijo de ese hijo esté liberado, el flush final best-effort de la sesión se asiente y el `AgentHandle` del hijo complete su liberación.

El asentamiento final espera a `ctx.sessions.flush(session)` pero ignora su booleano de participación porque un listener arbitrario no puede probar que un backend de persistencia almacenó el estado. El rechazo se registra sin hacer fallar la Activación, y el gestor igualmente libera el handle y suelta la propiedad; el estado persistido del hijo puede entonces faltar o estar obsoleto en una reanudación posterior. La descarga del gestor invoca un drenaje interno de todo el gestor que cierra la admisión y libera todo bosque vivo; `drainContinuableDescendants(parents)` cierra la admisión solo por debajo de los Agents exactos vivos propiedad del host y libera sus descendientes continuables mientras los bosques no relacionados siguen vivos. Ambos esperan las materializaciones ya admitidas en su ámbito, propagan la cancelación de arriba abajo, liberan los handles de los hijos primero y esperan cada rama seleccionada a pesar de los fallos individuales. Las Sessions durables de hijo sobreviven a ese desmontaje local al proceso.

```ts type-equiv
/** Attribution for a model coordinator's follow-up to one of its children. */
interface CoordinatorMessageSource {
  readonly kind: 'coordinator'
  /** A message another agent addressed to this one (`relay` context form). */
  readonly form: 'relay'
  /** Session id of the agent whose tool call produced the follow-up. */
  readonly senderSessionId: SessionId
}
```

```ts type-equiv
/** Options for following up with one continuable child. */
interface SubagentFollowupOptions {
  /** Durable attribution retained on the delivered message; it grants no authority. */
  readonly source: MessageSource
  /** Caller cancellation, owning the operation only until inbox acceptance. */
  readonly signal: AbortSignal
}
```

```ts type-equiv
/** Identities returned once a continuable child accepted its initial prompt. */
interface ContinuableStart {
  /** The durable child session id, stable across activations. */
  readonly childId: SessionId
  /** The accepted initial prompt's inbox message id. */
  readonly messageId: MessageId
}
```

Una contribución opcional de configuración del hijo continuable puede instalar capacidades con ámbito local después de la composición base del hijo y antes de la publicación de la Activación. El registro está ordenado y es transaccional: una configuración fallida o revocada revierte la Activación no publicada, la liberación del ámbito del hijo suelta cada instalación, los nuevos registros afectan a la siguiente Activación y la eliminación de un registro revoca de inmediato toda instalación residente.

`SubagentRuntime.reportFrom()` usa ese punto de extensión sin añadir una segunda cola ni un envoltorio de hijo con resultado. El Agent hijo vivo exacto autoriza la llamada; los llamadores no pueden nombrar un destinatario. El gestor deriva el único destinatario del `parentSession` durable del hijo, exige que ese Agent parent esté vivo, enmarca el contenido seleccionado como un mensaje de usuario `subagent-report` y devuelve el `MessageId` estable del mensaje. La entrega silenciosa (quiet) usa `Agent.inject()` y no despierta al parent; la entrega en el siguiente paso (next-step) usa `Agent.steer()`, despertando a un parent inactivo o uniéndose al límite de paso más cercano de un parent en ejecución. Ningún modo concluye el turno del hijo, y ninguna respuesta final se reporta implícitamente.

```ts type-equiv
/** Durable attribution for a continuable child's explicit parent report. */
interface SubagentReportMessageSource {
  readonly kind: 'subagent-report'
  /** A message another agent addressed to this one (`relay` context form). */
  readonly form: 'relay'
  /** Session id of the reporting child. */
  readonly senderSessionId: SessionId
}
```

```ts type-equiv
/** Deployment scheduling policy for accepted child reports. */
type SubagentReportDelivery = 'quiet' | 'next-step'
```

Reportar es decisión del propio hijo, así que el gestor mantiene una cuenta separada propia: cuando una Activación residente se asienta, entrega un aviso al parent directo durable del hijo describiendo cómo terminó esa época y llevando su contenido final de asistente. Esa entrega es incondicional para todo hijo cuyo id recibió un llamador, ocurre antes de la liberación de propiedad que permitiría juzgar asentado al parent, y llega a un parent residente a través de la misma contabilidad de admisión por despertar que un report. Un parent cuyo propio linaje ya se está desmontando lo recibe sin despertar, porque despertar a un Agent en quiescencia inicia un turno en lugar de poner trabajo en cola. Su procedencia es un tipo distinto, para que un transcript (transcripción) nunca presente una cuenta del runtime como algo que escribió el hijo.

```ts type-equiv
/**
 * Durable attribution for the runtime's own account of a continuable child
 * settling. Deliberately a different kind from
 * {@link SubagentReportMessageSource}: a report is content the child chose,
 * while this message is the manager stating what became of the child, and a
 * transcript that merged them would credit the child with words it never wrote.
 */
interface SubagentSettledMessageSource {
  readonly kind: 'subagent-settled'
  /** A runtime account shown without expanding the row (`notice` context form). */
  readonly form: 'notice'
  /** One-line account of how the child ended. */
  readonly summary: string
  /** Session id of the child that settled. */
  readonly senderSessionId: SessionId
}
```

```ts type-equiv
/** Options for one continuable child's report to its direct parent. */
interface SubagentReportOptions {
  /** Already-resolved parent scheduling policy. */
  readonly delivery: SubagentReportDelivery
  /** Caller cancellation, owning authorization and admission until acceptance. */
  readonly signal: AbortSignal
}
```

El provider participa solo en la preparación de la especificación de creación inicial, donde `spawn` y `fork` difieren. La especificación que devuelve transporta únicamente entradas de creación desprendidas específicas del provider —hoy, la semilla opcional de historia del parent— y ninguna operación de Agent, `AgentHandle`, entrega de prompt, resultado, liberación o reanudación. La reanudación en frío no pasa por ningún provider: el gestor pliega el descriptor genérico, llama a `ctx.agents.resume()` a través del mismo ámbito de propietario de activación y envía el turno en espera.

```ts type-equiv
/**
 * What the continuation manager asks a provider for while materializing one
 * continuable child's FIRST activation. The manager has already reserved the
 * durable child identity and owns every later operation, so this request
 * carries only what distinguishes a fresh child from one seeded with parent
 * history.
 */
interface ContinuableCreateRequest {
  /** The reserved durable child session id, for provider diagnostics. */
  readonly sessionId: SessionId
  /** The delegating parent agent whose history a seeding provider reads. */
  readonly parent: Agent
  /**
   * Caller cancellation, which owns preparation only until the manager accepts
   * the initial prompt into the child's inbox.
   */
  readonly signal: AbortSignal
}
```

```ts type-equiv
/**
 * A provider's detached contribution to one continuable child's creation. This
 * is DATA, never a capability: it carries no Agent, `AgentHandle`, prompt
 * delivery, result, disposal, or resume operation, because the continuation
 * manager owns the child's whole lifecycle after preparation.
 */
interface ContinuableCreateSpec {
  /**
   * Completed-turn prefix of the parent's log to seed the child session with,
   * or absent for a fresh child. Same durable contract as
   * `CreateAgentOptions.seed`: contiguous from seq 0, lossless JSON, balanced.
   */
  readonly seed?: readonly SessionEvent[]
}
```

El descriptor (`SubagentDescriptorData` en [descriptor.ts](../../packages/subagent/subagent/src/descriptor.ts)) es una identidad durable discriminada por modo para todo subagente respaldado por sesión. Ambos modos llevan el nombre del provider. Un descriptor `one-shot` lleva opcionalmente una `label` de presentación propiedad del llamador; un descriptor `continuable` exige la `description` de la delegación como su etiqueta durable de creación y además hace una instantánea del `agentOptions.provider`/`model` resueltos del hijo y de la `persona`/`toolFilter` opcionales para la reanudación en frío. Nunca hace una instantánea del objeto `AgentOptions` fusionable por extensión, de modo que un valor de extensión no relacionado no puede romper la continuación y una entrada de composición posterior es un cambio de versión deliberado. Omite `subagentDepth` (la reanudación en frío confía en el `delegationDepth` del encabezado persistido como suelo monótono) y `outputSchema` (el contrato de resultado de una ejecución o Activación, no la identidad durable).

Un provider one-shot local anexa el descriptor dentro del turno inicial del hijo, antes de su primera solicitud. El gestor de continuación anexa el descriptor después de cualquier linaje aportado por el provider y antes de admitir el prompt inicial; `header.seedLength` sigue siendo el límite del linaje de fork: la autoridad del descriptor en el momento de la reanudación lee el sufijo propio del hijo, mientras que la proyección de identidad que sirve el listado pliega `subagent/descriptor` con victoria del último, de modo que el descriptor propio del hijo anula el de un ancestro sembrado por fork. El evento es solo de registro: sin `surfaceOp`, nunca en la historia del modelo, y conservado a través de la compactación por el registro de solo anexión. Los descriptores malformados de la versión actual son corruptos; las versiones no soportadas no pueden clasificarse con este runtime.

## Enumeración durable: `listChildren()`, `listDescendants()` y sus entradas

`SubagentRuntime.listChildren(parentSessionId)` enumera los subagentes directos respaldados por sesión del parent a partir de la fusión con preferencia por lo vivo de `ctx.sessions.list()` y el opcional `ctx.sessionPersistence.list()` —sin servicio de consulta, y no se carga ni reanuda ningún Agent. Los candidatos son los hijos directos cuyo encabezado durable lleva `origin: 'subagent'`; el marcador clasifica la enumeración y la denegación genérica gruesa de rutas, pero no puede establecer un descriptor válido, la reanudabilidad ni la autorización —el plegado de proyección es dueño de la identidad, y el contrato de Activación es dueño de la reanudación. El `mode`/`label` de cada fila es el valor de la unidad de proyección `subagent` registrada, servido a través de una escalera de tres peldaños: la caché de marca de agua del registro para un hijo vivo (cero lecturas de registro); la caché opcional de puntos de control de la proyección para uno en frío (`cachedSnapshot` —una identidad que supera la compuerta de seq del sufijo propio es definitiva, porque un descriptor propio es inmutable una vez anexado—); si no, una lectura de `persistence.inspect()` plegada a través del registro (concurrencia acotada, recalculada por listado). La caché es un acelerador opcional puro: ausente, sirviendo el centinela `null` o sin la clave, fallando la compuerta de seq o fallando por error, cae silenciosamente al replegado autoritativo. El plegado es `subagent/descriptor` con victoria del último y sin canal de fallo: el descriptor propio del hijo anula el de un ancestro sembrado por fork, y un payload malformado o de versión desconocida se pliega a un centinela `null` serializable, tratado como ausencia de valor. El resultado es un `SubagentListEntry[]` en orden `createdAt` y luego id: una identidad servida produce una entrada `child` con `mode: 'one-shot' | 'continuable'` y `activity: 'running' | 'inactive'`; las entradas continuables llevan siempre `label`, mientras que las one-shot la llevan solo cuando el llamador del inicio aportó metadatos de presentación. Un candidato asentado cuyo plegado no sirvió identidad produce un diagnóstico `corrupt` —los descriptores ausentes, malformados y de versión desconocida se dejan deliberadamente sin distinguir (`unsupported` permanece en el tipo pero nunca se produce)—; un candidato en ejecución sin identidad se omite (la ventana de creación antes de que aterrice su descriptor); una inspección en frío fallida produce un diagnóstico `unavailable` que se reintenta en el siguiente listado, de modo que un hermano dañado no puede ocultar hijos sanos. `hasChildren` marca a un descendiente directo con origen de subagent durable, leído del mismo material fusionado. Activity solo refleja si el registro lógico está vivo en `ctx.sessions`, no el resultado ni la reanudabilidad. Sin persistencia, la enumeración es solo en vivo en lugar de un error —un hijo en frío tampoco podría reanudarse entonces. `listChildren()` lanza `SubagentError` con código `SUBAGENT_CONTROL_PROJECTIONS_UNAVAILABLE` cuando falta el registro `ctx.sessionProjections` y `SUBAGENT_CONTROL_SESSION_STORE_UNAVAILABLE` cuando falta el almacén de sesiones, ambos comprobados antes de cualquier lectura, de modo que un despliegue sin hijos sigue fallando de forma determinista; la herramienta de listado exige `ctx.subagents` y `ctx.agents` al cargar el plugin. Un consumidor del servicio, como una interfaz de usuario, puede mostrar ambos modos y elegir una alternativa one-shot sin etiqueta, mientras que el adaptador `list_agents` orientado al modelo (el plugin `/list-agents` cargable por separado de [dsh-tool-subagent-control](../../packages/subagent/tool-subagent-control)) conserva solo las entradas continuables y refina el estado a través del registro vivo de Agent hasta su propio vocabulario `running`/`idle`/`ready`, cuyo `ready` nombra a un hijo solo de almacenamiento como reanudable en lugar de terminal. El listado no consulta el mapa de Activations del gestor de continuación, el registro de Agent ni la disponibilidad de providers; `send_message` sigue siendo la operación autoritativa en el momento de la entrega, y un hijo continuable en ejecución listado puede seguir rechazando la entrega como conflicto de propiedad. La justificación del camino de lectura vive en [el Agent Note de proyección de identidad de listado](../../.agents/notes/implemented/architecture/2026-08-06-subagent-list-identity-projection.es.md).

`SubagentRuntime.listDescendants(rootSessionId)` aplica el mismo corpus con preferencia por lo vivo y la misma interpretación respaldada por proyección a todo el árbol de descendientes de la raíz en preorden estable. Las sesiones ordinarias y los hijos one-shot siguen siendo nodos de recorrido, de modo que se descubren los descendientes continuables por debajo de ellos; solo los candidatos `origin: 'subagent'` producen filas. Cada hijo o diagnóstico devuelto añade su posición del encabezado durable enumerado, mientras que una inspección en frío revalida ese ciclo de vida completo antes de servir la identidad:

```ts type-equiv
/**
 * One entry of a descendant listing: the interpreted subagent facts plus its
 * position in the complete session tree. `parentId` is the durable direct
 * parent from the enumerated header, and `depth` counts edges from the root.
 */
type SubagentDescendantListEntry = SubagentListEntry & {
  /** Durable direct parent of this candidate in the enumerated tree. */
  readonly parentId: SessionId
  /** Edge distance from the requested root; direct children are `1`. */
  readonly depth: number
}
```


## El resultado terminal: `SubagentResult`

El resultado de una ejecución one-shot, resuelto por `SubagentRun.result`. `structured` está presente solo después de que un `outputSchema` solicitado se haya satisfecho con éxito; solicitar un schema no lo garantiza, y un provider puede devolver `stopReason: 'error'` cuando el hijo falla o termina sin una captura válida. Un provider puede adjuntar un `diagnostic` seguro y no asistente a un resultado no `completed`; el provider elimina entradas de herramientas, contenidos de archivos, valores de entorno, credenciales y payloads de protocolo crudos, y limita el valor completo a 4096 bytes UTF-8 antes de que los consumers lo presenten por separado de `output`. Un `stopReason` no `completed` significa que `output` puede ser parcial —el consumer lo mapea a un resultado de herramienta `isError` en lugar de reportar la salida parcial como éxito.

```ts type-equiv
/**
 * The terminal outcome of a subagent run, resolved by {@link SubagentRun.result}.
 */
interface SubagentResult {
  /**
   * The child's final assistant output is the content of its last non-empty
   * assistant message. Empty-content messages, including usage-only messages,
   * are skipped. Without a non-empty message, the output is its accumulated
   * assistant text stream, or `[]` when the child produced neither.
   */
  readonly output: ContentBlock[]
  /**
   * The structured result after a requested `outputSchema` was successfully
   * satisfied. Requesting a schema does not guarantee presence: a provider can
   * end with `stopReason: 'error'` when the child fails or finishes without a
   * valid capture. The structured value is validated against the requested
   * output schema by the provider; `unknown` here because the seam is
   * schema-agnostic.
   */
  readonly structured?: unknown
  /**
   * Provider-authored, non-assistant failure detail for a non-`completed`
   * result. Providers keep this text free of tool inputs, file contents,
   * environment values, credentials, and raw protocol payloads, and limit it
   * to 4096 UTF-8 bytes. Consumers present it separately from {@link output}.
   */
  readonly diagnostic?: string
  /** Why the run ended. A non-`completed` reason means `output` may be partial. */
  readonly stopReason: SubagentStopReason
}
```

`SubagentStopReason` es una [unión derivada fusionable por extensión](core.es.md#the-map--derived-union-pattern) —un backend puede añadir variantes, así que los consumers ramifican sobre los casos conocidos y tratan una razón terminal desconocida como un fallo:

```ts type-equiv
/**
 * Why a subagent run ended. Merge-extensible (a backend may add variants);
 * consumers branch on the known cases and fall through `default`. The known
 * cases mirror the harness turn-end vocabulary so the tool layer can map a
 * non-`completed` result to an `isError` tool result.
 */
interface SubagentStopReasonMap {
  /** The child finished its turn normally. */
  completed: 'completed'
  /** Cancelled through the request signal or disposal. */
  aborted: 'aborted'
  /** Model or transport failure. */
  error: 'error'
  /** The child hit its token ceiling before finishing. */
  'max-tokens': 'max-tokens'
  /** The child declined the task. */
  refusal: 'refusal'
}
```

## Una ejecución one-shot: `SubagentRun`

`SubagentRun` es el handle propiedad del consumer de un hijo one-shot publicado —una delegación desechable en primer plano con un solo resultado, nunca un handle durable de hijo. El envío del prompt, el trabajo de turnos y los fallos de infraestructura posteriores a la publicación pertenecen a `result`. Los consumers esperan ese resultado y siempre liberan (dispose) la ejecución para alcanzar la quiescencia. Los fallos del hijo se resuelven con una razón de parada no completada; solo los fallos de infraestructura no representables se rechazan. Una ejecución no tiene steering ni reanudación: las conversaciones continuables no tienen ejecución alguna, porque el gestor de continuación sostiene su `AgentHandle` directamente y ordena cada turno a través del buzón propio del hijo.

```ts type-equiv
/**
 * ONE-SHOT child handle returned after publication. Prompt submission, turn
 * work, and infrastructure faults after that boundary belong to {@link result}.
 * Consumers await that result and must always {@link dispose} to cancel
 * remaining work and reach quiescence. A run is one disposable foreground
 * delegation with one result; continuable conversations have no run — the
 * continuation manager holds their `AgentHandle` directly and orders every
 * turn through the child's own inbox.
 */
interface SubagentRun {
  /**
   * Parent-scoped run id. For a local run, this MUST equal the published child
   * session id, whose `parentSession` records `request.parent.session.id`; a
   * remote provider mints an id unique in the parent namespace.
   */
  readonly id: SessionId
  /**
   * The exact published in-process child, or `undefined` for a remote run.
   * When present, its id is {@link id}; the provider retains no ownership
   * implication beyond the run's ordinary {@link dispose} contract.
   */
  readonly localAgent: Agent | undefined
  /**
   * Resolves with the child's terminal {@link SubagentResult} when the run
   * settles. Does NOT reject on a child-level failure — a model/transport
   * failure resolves with `stopReason: 'error'` so the consumer maps it to an
   * `isError` tool result. Rejects on an infrastructure fault the seam cannot
   * represent as a stop reason.
   */
  readonly result: Promise<SubagentResult>
  /**
   * Cancel remaining work, reach child quiescence, and release resources.
   * Idempotent.
   */
  dispose(): Promise<void>
}
```

Una ejecución one-shot local DEBE publicar un agent/sesión hijo ordinario antes de que `start()` se cumpla, devolver ese id de sesión de hijo como `SubagentRun.id`, exponer al hijo exacto como `localAgent`, registrar `request.parent.session.id` en el encabezado `parentSession` del hijo y anexar el descriptor resuelto dentro del turno inicial del hijo antes de su primera solicitud. La propiedad del runtime puede colocar al hijo bajo el ámbito del parent, del provider o de la raíz. Un provider remoto, en cambio, devuelve un id de ciclo de vida con ámbito de parent y `localAgent: undefined`; sin una Session de hijo local, queda ausente de la enumeración durable.

## El contrato del provider: `SubagentProvider`

Cada provider es un transporte de agent hijo con nombre, y varios providers pueden coexistir. El servicio valida las capacidades de tiempo de arranque solicitadas antes de `start()`, y rechaza un inicio continuable en un provider sin `prepareContinuable`. `inheritsParentContext` describe solo la siembra de conversación (`fork`: true; `spawn` y `acp`: false), permitiendo que los consumers generen un texto orientado al modelo preciso sin implicar herramientas, servicios ni autoridad heredados.

```ts type-equiv
/**
 * One registered transport for running child agents. Providers are trusted
 * same-process implementations; callers treat descriptors and returned values
 * as borrowed immutable data. The service may call one provider concurrently
 * for distinct children. Providers isolate operation-local mutable state; a
 * shared capacity controller may delay an operation but must not couple its
 * settlement or cleanup to a sibling.
 */
interface SubagentProvider {
  /** Unique registry name (e.g. `spawn`, `fork`, `acp`). */
  readonly name: string
  /** The start-time features this provider supports (see {@link SubagentCapabilities}). */
  readonly capabilities: SubagentCapabilities
  /**
   * Whether the child sees the parent's completed-turn prefix. This is descriptive, not a
   * service-validated start capability: the model-facing tool derives truthful wording from it.
   * It says nothing about tool registration, injected services, or authority inheritance.
   */
  readonly inheritsParentContext: boolean
  /**
   * Establish a ONE-SHOT child and return its handle after publication.
   * The service has already validated that every requested start-time
   * capability is supported and resolved `request.descriptor`, so a
   * session-backed implementation appends that descriptor inside the child's
   * initial turn. Before fulfillment, the provider owns setup and cleans any
   * unpublished partial resources before rejecting. Ownership transfers on
   * fulfillment; subsequent turn or infrastructure failure settles through
   * the returned run. Distinct starts may overlap; cancellation, failure,
   * result settlement, and disposal remain independent for each run.
   */
  start(request: ResolvedSubagentStartRequest): Promise<SubagentRun>
  /**
   * OPTIONAL (continuable-creation capability): contribute the detached
   * creation inputs that distinguish this provider's continuable children —
   * only whether the child session is seeded with parent history. Method
   * presence IS the capability: the service rejects continuable starts on
   * providers without it, while a provider that has it may still serve
   * ordinary one-shot delegations.
   *
   * This is the provider's ONLY participation in a continuable child. The
   * continuation manager owns identity reservation, composition, Agent
   * creation, prompt delivery, cold resume, ownership, and disposal, so a
   * provider never sees the child's Agent, handle, turns, or teardown.
   * Distinct preparations may overlap; each follows its own signal and returns
   * data belonging only to `request.sessionId`.
   */
  prepareContinuable?(request: ContinuableCreateRequest): Promise<ContinuableCreateSpec>
}
```

El `start()` del provider se cumple con una ejecución publicada. El servicio acuña un `runId` único, hace una instantánea de `local` desde el `localAgent` exacto del provider, observa el resultado, emite `subagent/start` y devuelve la misma ejecución; un rechazo de `start()` implica la limpieza de los recursos no publicados y no emite ningún par de ciclo de vida, mientras que un rechazo de resultado posterior a la publicación cierra el par emitido. Cada Activación continuable emite el mismo par solo de observación para su época de residencia, de modo que una reanudación en frío es una época nueva con su propio `runId`. El `subagent/end` emparejado lleva la misma identidad y la salida final o el fallo de infraestructura. Ambos eventos son solo de observación y contienen las excepciones de los listeners. Su campo `provider` nombra al provider que inició la ejecución o la época de Activación; no afirma que el provider siga registrado cuando se emite el evento.

## Backends en proceso: profundidad y semilla

Los backends spawn y fork crean un agent one-shot ordinario a través de `parent.ctx`, pasan la cancelación a la creación del núcleo y liberan a través de `AgentHandle`; un hijo continuable, en cambio, lo crea el gestor de continuación a través de su propio ámbito de propietario de activación. La retirada de un provider bloquea los nuevos inicios sin revocar las ejecuciones aceptadas. Cada hijo recibe un ámbito plano nuevo en lugar de heredar los registros del parent. La profundidad y la siembra de fork reutilizan el vocabulario existente de agent y sesión:

- **Profundidad de delegación**: es el durable `SessionHeader.delegationDepth` más el campo de runtime fusionable por extensión `AgentOptions.subagentDepth`; la ausencia significa profundidad cero de nivel superior, y el mayor de los valores presentes es el autoritativo. El seam es dueño de ambos campos —el loop ni los fija ni los lee—, de modo que un hijo en proceso persiste profundidad del parent + 1, la reanudación en frío no puede bajarla y todo inicio rechaza una profundidad derivada fuera del dominio de los enteros seguros o por encima de un tope absoluto `request.maxDepth` definido.
- **Siembra de fork**: usa [`CreateAgentOptions.seed`](core.es.md#creation-and-ownership) (un prefijo `SessionEvent[]` enhebrado a través de `AgentLoop.createAgent` → `ctx.sessions.prepare({ seed })`, la misma primitiva que usa `ctx.agents.resume()`). El backend fork pasa un *prefijo equilibrado de turnos completados* del registro del parent —los eventos del parent hasta su último `turn/end` inclusive—, de modo que la semilla es contigua desde 0 y la reproducción de los [invariantes](../../packages/runtime-diagnostics/invariants) la acepta (el turno en vuelo, no equilibrado, queda excluido).

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxsubagents--subagentruntime"></a>

### `ctx.subagents` — `SubagentRuntime`

Named provider registry with one-shot runs, durable discovery, and continuable-child operations.

```ts cordis-catalog
/**
 * Establish one durable continuable child and deliver its initial prompt.
 * Resolves when the child's inbox accepts that prompt, without waiting for the
 * turn to start or for the message to reach the Session log; any earlier
 * failure rejects with no ids and rolls back the child entirely.
 * @param spec - provider, delegation request, and caller cancellation.
 * @returns the durable child id and the accepted prompt's message id.
 * @throws when continuation services are unavailable or materialization fails.
 */
async startContinuable(spec: ContinuableStartSpec): Promise<ContinuableStart>

/**
 * Deliver one later message to a continuable child as its next FIFO turn. A
 * resident child's Agent inbox accepts it directly (waking a `waiting`
 * Activation), while an absent one is cold-resumed from its persisted
 * Session. The Agent inbox is the only queue, so every accepted message has
 * one observable order.
 * @param parent - the exact live direct parent authorizing this delivery.
 * @param childId - durable child session id.
 * @param content - user-role content to deliver.
 * @param options - the message source fields and caller cancellation, which stops the
 *   operation only before inbox acceptance.
 * @returns the accepted message's inbox id.
 * @throws when continuation services are unavailable, parent authority is
 *   rejected, or the message was not admitted.
 */
async followup( parent: Agent, childId: SessionId, content: ContentBlock[], options: SubagentFollowupOptions, ): Promise<MessageId>

/**
 * Interrupt one live continuable child's current turn under a human parent
 * address or an exact live ancestor Agent. Fire-and-return: the cancel
 * signal is issued before this returns, but the target may keep running
 * until it observes the signal. Unclaimed pending inbox work, the Activation,
 * and published descendants are preserved; claimed work is not requeued.
 * Once the interrupted driver is idle, a waking send resumes the parked FIFO
 * queue. An absent target — including a one-shot or unknown id —
 * is an accepted no-op, as is a manager-less composition, which cannot own a
 * live Activation.
 * @param targetSessionId - the durable child session id to interrupt.
 * @param authority - the human parent address or exact live ancestor Agent.
 * @throws {SubagentError} `UNAUTHORIZED` when the authority does not own the
 *   live target.
 */
interrupt(targetSessionId: SessionId, authority: SubagentInterruptAuthority): void

/**
 * Deliver selected content from one live continuable child to its durable
 * direct parent. The child is the authority credential; callers cannot name a
 * recipient. Reporting does not conclude the child's turn or Activation.
 * @param child - exact live reporting child.
 * @param content - selected model-facing content.
 * @param options - parent scheduling and pre-acceptance cancellation.
 * @returns the stable identity of the parent-accepted message.
 * @throws when continuation services are unavailable, sender authorization
 *   fails, or the direct parent is not live.
 */
async reportFrom( child: Agent, content: ContentBlock[], options: SubagentReportOptions, ): Promise<MessageId>

/**
 * Compose one deployment capability into every continuable child's
 * unpublished creation context on fresh creation and cold resume. Grants wait
 * for the next Activation; removing the contribution revokes every resident
 * installation immediately.
 * @param contribution - synchronous child-scope installer.
 * @returns the exact Cordis effect disposer.
 */
registerContinuableSetup(contribution: ContinuableSetupContribution): () => void

/**
 * Close continuable admission below exact live parent Agents, stop only their
 * visible descendant Activations synchronously, then await admitted scoped
 * materializations and release those forests child-first. The scoped cutoff
 * lasts until each exact parent leaves the registry; unrelated parent trees
 * remain live.
 * @param parents - exact host-owned parent Agents entering teardown.
 * @returns once every retained descendant Activation released its `AgentHandle`.
 * @throws an aggregate error after all branches settle when any failed.
 */
async drainContinuableDescendants(parents: readonly Agent[]): Promise<void>

/**
 * Release selected resident continuable direct children of one exact live
 * parent. Other children of the same parent remain admitted and resident.
 * Absent targets and a manager-less composition are accepted no-ops.
 * @param parent - exact live direct parent authorizing the selected release.
 * @param childIds - durable direct-child ids to release when resident.
 * @returns once every selected Activation released its `AgentHandle`.
 * @throws {SubagentError} `UNAUTHORIZED` when a resident target belongs to a
 *   different parent or the supplied parent identity is stale.
 */
async drainContinuableChildren(parent: Agent, childIds: readonly SessionId[]): Promise<void>

/**
 * Enumerate the parent's direct session-backed subagents without loading or
 * resuming an Agent and without any query service: the listing merges the live
 * session store with optional session persistence (live-preferred) and
 * serves each child's durable mode/label from the registered `subagent`
 * projection unit down a three-rung ladder — the registry's watermark
 * snapshot for a live child; for a cold one, a durable projection-cache
 * row when the optional cache serves an own-suffix identity (its `seq`
 * gate proves the value postdates the fork seed, where a child's own
 * descriptor is immutable once appended), else one persistence inspection
 * folded through the registry. The
 * projection fold is the single classification authority; per-child
 * diagnostics relay a fold that served no identity or a failed inspection,
 * never a list-time descriptor parse. Absent persistence, enumeration is
 * live-only (a cold child cannot be resumed then either, so its absence is
 * capability absence, not an error). This service consults no Agent
 * registrations, Activations, or providers.
 *
 * Every persistence read receives `signal`, and the listing rechecks
 * cancellation around each of those awaits. Read rejections that settle
 * after an abort become a stable `SubagentError` with code `CANCELLED`.
 * @param parentSessionId - parent session whose direct children are listed.
 * @param signal - caller-owned cancellation forwarded to persistence reads
 *   and observed around every read await.
 * @returns children and per-child diagnostics ordered by `createdAt`, then id.
 * @throws {@link SubagentError} when the projection registry or the session
 *   store is not mounted, or the caller cancels the listing.
 */
listChildren(parentSessionId: SessionId, signal?: AbortSignal): Promise<SubagentListEntry[]>

/**
 * Enumerate the root's complete session-backed subagent tree in stable
 * pre-order from one live-preferred corpus, without loading or resuming an
 * Agent. Ordinary sessions and one-shot children remain traversal nodes so
 * continuable descendants below them are discovered; each returned entry
 * adds its durable `parentId` and root-relative `depth`. Identity resolution,
 * diagnostics, optional persistence, and cancellation follow the same
 * projection-backed contract as {@link listChildren}.
 * @param rootSessionId - session whose complete descendant tree is listed.
 * @param signal - caller-owned cancellation forwarded to persistence reads
 *   and observed around every read await.
 * @returns children and per-candidate diagnostics with tree position, in
 *   stable pre-order.
 * @throws {@link SubagentError} under the same conditions as {@link listChildren}.
 */
listDescendants(rootSessionId: SessionId, signal?: AbortSignal): Promise<SubagentDescendantListEntry[]>

/**
 * Register a provider under its name. Registration is effect-scoped and HMR
 * safe; removing a provider blocks new starts but does not revoke runs that
 * were already returned to their holders.
 * @param provider - the trusted provider implementation.
 * @returns the exact Cordis effect disposer.
 */
registerProvider(provider: SubagentProvider): () => void

/**
 * Look up a provider by name.
 * @param name - the provider name.
 * @returns the provider, or undefined when absent.
 */
getProvider(name: string): SubagentProvider | undefined

/**
 * List registered provider names in insertion order.
 * @returns the registered names.
 */
list(): string[]

/**
 * Establish a published child on the named provider. Capability and semantic
 * checks run before delegation. Provider ownership lasts until its promise
 * fulfills; a rejection therefore has no run for the caller to dispose and
 * emits no run lifecycle events. Post-publication turn and infrastructure
 * failures settle through the returned run.
 * @param name - the provider to use.
 * @param request - child label, prompt, parent, signal, and optional capabilities.
 * @returns the published holder-owned run.
 */
async start(name: string, request: SubagentStartRequest): Promise<SubagentRun>
```

Types: [Agent](core.es.md) · [ContentBlock](llm-streaming.es.md) · [MessageId](llm-streaming.es.md) · [SessionId](core.es.md)

Source: [`packages/subagent/subagent/src/index.ts`](../../packages/subagent/subagent/src/index.ts)

<a id="subagent-events"></a>

### `subagent/*` events

<a id="subagentend--emit"></a>

#### `subagent/end` — emit

A published child settled. Scope-filtered dispatch uses the same delegating parent carrier as `subagent/start`, so the lifecycle pair reaches the same scoped audience.

```ts cordis-catalog
/**
 * A published child settled. Scope-filtered dispatch uses the same delegating
 * parent carrier as `subagent/start`, so the lifecycle pair reaches the
 * same scoped audience.
 * @param info - the run identity and terminal outcome.
 * @dshScopeScan unsupported
 * @mode emit
 */
'subagent/end'(this: Scoped<SubagentRuntime>, info: SubagentRunEndInfo): void
```

Types: [Scoped](scope.es.md)

Source: [`packages/subagent/subagent/src/index.ts`](../../packages/subagent/subagent/src/index.ts)

<a id="subagentprovider-added--emit"></a>

#### `subagent/provider-added` — emit

A provider became resolvable in the registry.

```ts cordis-catalog
/**
 * A provider became resolvable in the registry.
 * @param provider - the registered provider.
 * @mode emit
 */
'subagent/provider-added'(provider: SubagentProvider): void
```

Source: [`packages/subagent/subagent/src/index.ts`](../../packages/subagent/subagent/src/index.ts)

<a id="subagentprovider-removed--emit"></a>

#### `subagent/provider-removed` — emit

A provider left the registry. Accepted runs remain holder-owned.

```ts cordis-catalog
/**
 * A provider left the registry. Accepted runs remain holder-owned.
 * @param name - the provider name that no longer resolves.
 * @mode emit
 */
'subagent/provider-removed'(name: string): void
```

Source: [`packages/subagent/subagent/src/index.ts`](../../packages/subagent/subagent/src/index.ts)

<a id="subagentstart--emit"></a>

#### `subagent/start` — emit

A provider established a published child. For in-process providers, `ctx.agents.get(info.id)` resolves during this notification. Scope-filtered dispatch keys the carrier by the delegating parent, so a parent-scoped listener observes only its own delegations. Paired with `subagent/end`.

```ts cordis-catalog
/**
 * A provider established a published child. For in-process providers,
 * `ctx.agents.get(info.id)` resolves during this notification.
 * Scope-filtered dispatch keys the carrier by the delegating parent, so a
 * parent-scoped listener observes only its own delegations. Paired with
 * `subagent/end`.
 * @param info - the provider and published child identity.
 * @dshScopeScan unsupported
 * @mode emit
 */
'subagent/start'(this: Scoped<SubagentRuntime>, info: SubagentRunInfo): void
```

Types: [Scoped](scope.es.md)

Source: [`packages/subagent/subagent/src/index.ts`](../../packages/subagent/subagent/src/index.ts)
<!-- END GENERATED cordis-surface -->
