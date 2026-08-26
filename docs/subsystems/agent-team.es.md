# Agent Teams

[English](agent-team.md) | Español

Tipos compartidos por el dominio experimental de Team de raíz implícita, las herramientas de modelo y los adaptadores de host. El [Agent Teams Agent Note](../../.agents/notes/implemented/feature/2026-08-05-agent-teams.es.md) es dueño de las decisiones de identidad, mailbox, tarea y checkout compartido; esta página registra las formas durables literales de [`packages/experimental/agent-team/src/types.ts`](../../packages/experimental/agent-team/src/types.ts).

## Identidad y roster

`TeamId` es el `SessionId` raíz bajo una [marca](core.es.md#branded-ids) distinta. `TeamTaskId` es local al Team y se asigna de forma monótona como `task-<n>`; `TeamMessageId` es globalmente aleatorio. El id de Session de un teammate sigue siendo su identidad persistente, mientras que `name` es una etiqueta inmutable de modelo/UI.

```ts type-equiv
/** Whole durable value written on every teammate lifecycle change. */
interface TeamMemberSnapshot {
  readonly id: SessionId
  readonly name: string
  readonly description: string
  readonly provider: string
  readonly context: 'fresh' | 'fork'
  readonly phase: TeamMemberPhase
  readonly error?: string
}
```

Todo miembro comienza en `provisioning` y alcanza exactamente una fase de roster terminal, `active` o `failed`. El estado de runtime `running`/`idle`/`inactive` se deriva por separado y nunca reescribe este registro.

## Mailbox durable

La sesión Lead almacena primero el mensaje en cola completo. El recibo de destino solo se reconoce después de que su elemento pendiente de inbox o su mensaje de usuario registrado sea durable, dejando queued-minus-delivered como mailbox de recuperación.

```ts type-equiv
/** One peer message retained until its target Session records it. */
interface TeamMessageSnapshot {
  readonly id: TeamMessageId
  readonly senderId: SessionId
  readonly senderName: string
  readonly targetId: SessionId
  readonly delivery: 'quiet' | 'wakeup'
  readonly content: ContentBlock[]
}
```

La sesión de destino conserva la identidad del mensaje y la atribución del remitente tanto en el elemento pendiente de inbox como en el mensaje de usuario final. Plegar esa fuente entre inbox e historial es la clave de desduplicación del lado del destino; el encuadre visible para el modelo repite el id y el remitente.

```ts type-equiv
/** Source retained by the target Session for durable mailbox de-duplication. */
interface TeamMessageSource {
  readonly kind: 'team-message'
  readonly teamId: TeamId
  readonly messageId: TeamMessageId
  readonly senderId: SessionId
  readonly senderName: string
}
```

## DAG de tareas compartido

Cada evento de tarea almacena una instantánea completa. `revision` es el valor de comparar-y-asignar y se incrementa en uno por mutación. Las aristas `blockedBy` deben nombrar tareas no eliminadas y mantener el grafo acíclico. `writeScopes` son prefijos de ruta normalizados de carácter consultivo, no bloqueos.

```ts type-equiv
/** Whole durable task snapshot; every mutation increments {@link revision}. */
interface TeamTaskSnapshot {
  readonly id: TeamTaskId
  readonly revision: number
  readonly subject: string
  readonly description: string
  readonly status: TeamTaskStatus
  readonly ownerId?: SessionId
  readonly blockedBy: TeamTaskId[]
  readonly writeScopes: string[]
}
```

`pending` es no iniciado o liberado, `in_progress` lleva un propietario, `completed` satisface los bloqueadores y `deleted` es una tombstone retenida. Las vistas añaden el nombre del propietario, la disposición y advertencias de solapamiento de write-scopes sin cambiar la instantánea durable.

## Reproducción

`foldTeam()` reproduce una sesión raíz en el roster, el tablero de tareas y el mailbox queued-minus-delivered que lee toda operación de Team. Selecciona registros por `TeamId`, así que los eventos heredados por un fork ordinario conservan el id del ancestro y nunca entran en el estado de la nueva raíz. El `seq` y el `time` de los eventos de sesión siguen siendo el registro de orden y tiempo; las instantáneas de Team no los duplican. Las lecturas de roster y de tareas llegan a los llamadores como vistas que añaden el nombre del propietario, la disposición y advertencias de write-scopes, mientras que el correo pendiente permanece interno a la entrega y la recuperación. El [README](../../packages/experimental/agent-team/README.es.md) del paquete es dueño del comportamiento de operación, autorización, recuperación y límites.

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxagentteams--teamservice"></a>

### `ctx.agentTeams` — `TeamService`

Agent Teams service backed by the exact live Lead Session log.

```ts cordis-catalog
/**
 * Resolve one exact live Agent's Team role.
 * @param agent - exact live Agent used as the authority credential.
 * @returns its root, Team identity, role, and model-facing name.
 */
membership(agent: Agent): TeamMembership

/**
 * List the runtime-enriched roster visible to one Team member.
 * @param agent - exact live Team member.
 * @returns Lead and teammate rows in creation order.
 */
listMembers(agent: Agent): TeamMemberView[]

/**
 * Create one named, continuable direct child of the Team Lead.
 * @param caller - exact live Lead Agent.
 * @param request - immutable name, description, prompt, context mode, provider, and cancellation.
 * @returns the active roster row.
 */
async spawnTeammate(caller: Agent, request: SpawnTeammateRequest): Promise<SpawnTeammateResult>

/**
 * Queue one durable peer message, then attempt immediate delivery.
 * @param caller - exact live sending Team member.
 * @param request - target name, content, scheduling mode, and pre-queue cancellation.
 * @returns durable message identity and immediate-delivery observation.
 */
async sendMessage(caller: Agent, request: SendTeamMessageRequest): Promise<SendTeamMessageResult>

/**
 * Create one unowned pending task in the Team Lead log.
 * @param caller - exact live Team member creating the task.
 * @param request - task text, blockers, and advisory write scopes.
 * @returns the revision-one task view.
 */
async createTask(caller: Agent, request: CreateTeamTaskRequest): Promise<TeamTaskView>

/**
 * Return one task, including a deleted tombstone.
 * @param caller - exact live Team member reading the task.
 * @param id - Team-local task identity.
 * @returns the latest task value and derived readiness diagnostics.
 */
getTask(caller: Agent, id: TeamTaskId): TeamTaskView

/**
 * List current non-deleted tasks in numeric creation order.
 * @param caller - exact live Team member reading the board.
 * @returns detached current task views.
 */
listTasks(caller: Agent): TeamTaskView[]

/**
 * Compare-and-set one authorized task transition.
 * @param caller - exact live Team member authorizing the mutation.
 * @param request - task identity, expected revision, action, and action fields.
 * @returns the committed next task revision.
 */
async updateTask(caller: Agent, request: UpdateTeamTaskRequest): Promise<TeamTaskView>

/**
 * Wait for the next Team-domain or member-status change.
 * @param caller - exact live Team member waiting for activity.
 * @param timeoutMs - bounded wait duration from ten seconds through one hour.
 * @param signal - caller cancellation for the wait only.
 * @returns one observed change or a timeout result.
 */
async waitForChange(caller: Agent, timeoutMs: number, signal: AbortSignal): Promise<TeamWaitResult>

/**
 * Interrupt one live teammate turn without clearing its pending inbox.
 * @param caller - exact live Lead Agent.
 * @param targetName - durable teammate name.
 * @returns the target status sampled before cancellation.
 */
interrupt(caller: Agent, targetName: string): { previousStatus: 'running' | 'idle' | 'inactive' }

/**
 * Resolve a caller without throwing, used by scoped-tool installation and observers.
 * @param agent - candidate exact live Agent.
 * @returns Team membership, or undefined for non-Team subagents and stale identities.
 */
tryMembership(agent: Agent): TeamMembership | undefined
```

Types: [Agent](core.es.md)

Source: [`packages/experimental/agent-team/src/index.ts`](../../packages/experimental/agent-team/src/index.ts)
<!-- END GENERATED cordis-surface -->
