# Sesiones

[English](session.md) | Español

El modelo en memoria, basado en event sourcing, de [dsh-session](../../packages/core/session). Una `Session` es un **log de solo añadidura** de `SessionEvent` tipados: la única fuente de verdad para todo el historial de interacción de un agent (agente). El historial de mensajes del LLM (modelo de lenguaje de gran tamaño) se *deriva* del log y nunca se almacena por separado; la reproducción es una re-derivación a partir de los mismos eventos. Cómo se hace **duradero** el log (el seam de persistencia, los backends, la recuperación ante caídas) es la preocupación hermana que se trata en [persistence.md](persistence.es.md).

Fuente: [`packages/core/session/src/types.ts`](../../packages/core/session/src/types.ts)

## `SessionEventMap` — el vocabulario de eventos

Los tipos de evento de solo añadidura. Extensibles por fusión de declaraciones: un plugin declara tipos de evento adicionales fusionando declaraciones — p. ej. el [seam de compactación](compaction.es.md) añade `compaction/start` / `compaction/summary` / `compaction/end`, y `@deepseek-ai/dsh-hook-protocol` añade registros de solo log `hook/invoked` / `hook/result` para un puente de hooks. Al igual que `compaction/*`, estos NO son `SurfaceEventType` (no tienen `surfaceOp`). El [catálogo de eventos del log de persistencia](../persistence-catalog.es.md) generado enumera cada miembro — de core y fusionados — con su payload, su insignia de surface (superficie) y su lugar de declaración.

```ts type-equiv
/** A user-role specialization of the one shared message representation. */
interface UserMessage extends Message {
  readonly role: 'user'
}
```

```ts type-equiv
/**
 * The merge-extensible, append-only source of truth for an agent interaction.
 * Message history is derived from this log. Every event is lossless JSON and
 * sequence numbers stay contiguous, including raw chunks, so persistence can
 * store the canonical log verbatim.
 */
interface SessionEventMap {
  /**
   * Opens turn `turn` before the loop claims queued input or runs pre-step.
   * Rejection, empty input, cancellation, or failure may close it with no
   * step; otherwise the following identified `user/message` event or batch
   * records the messages entering the step.
   */
  'turn/start': { turn: number }
  /**
   * Closes turn `turn` with the {@link TurnEndReason} that ended it. A turn
   * with no entered step has no `step/start` or `step/end`. The loop does not await a
   * flush at turn boundaries: `dsh-session-checkpoint-policy` owns the
   * per-request durability checkpoint, and consumers that read storage after
   * `whenIdle()` flush themselves. Success commits the turn; rejection is
   * reported live and does not prevent later work.
   */
  'turn/end': { turn: number; reason: TurnEndReason }
  /** Opens step `step` of turn `turn` — one model call plus the tool executions it requested. */
  'step/start': { turn: number; step: number }
  /** Closes step `step` of turn `turn`. */
  'step/end': { turn: number; step: number }
  /**
   * A user-role message on the model-visible surface: a direct human prompt
   * (the queued message claimed for this turn), a synthetic `agent.inject()`
   * context (file-change notices, subdir AGENTS.md, skill content, cron
   * notifications, …), or an entered goal continuation round. All three
   * project their `content` verbatim; `source` tells them apart.
   */
  'user/message': UserMessage
  /** Raw stream chunk — token-level replay fidelity. */
  'assistant/chunk': { turn: number; step: number; chunk: StreamChunk }
  /**
   * Assembled assistant message for one step (derived history uses this).
   * Carries the step's `usage` when the adapter reported token accounting, so
   * the model output and its accounting travel together (there is no separate
   * usage record). `usage` is absent when the adapter reported none. A turn
   * cancelled mid-stream finalizes its delivered text/reasoning prefix as this
   * event with `interrupted: true`; undispatched tool calls are absent. The
   * marker distinguishes that prefix without re-deriving interruption from turn
   * boundaries. An aborted turn with no such event streamed no visible content.
   */
  'assistant/message': { turn: number; step: number; message: AssistantMessage; usage?: TokenUsage; interrupted?: true }
  /**
   * The model requested one tool invocation: `name` with the raw `arguments`
   * JSON string exactly as the model produced it (unparsed). `callId` pairs the
   * call with its `tool/result`.
   */
  'tool/call': { turn: number; step: number; callId: CallId; name: string; arguments: string }
  /**
   * A completed tool call's model-facing result, optional internal failure
   * identity, and optional tool-private `meta` presentation payload. `meta` is
   * opaque to the core (the producing tool owns its shape and reads it back in
   * `presentResult`) but MUST be JSON-serializable: `Session.append`
   * runtime-validates all event data with `isJsonValue`, so a non-serializable
   * `meta` is rejected at the source, and the durable log reproduces the
   * identical card on replay. Absent
   * unless the tool attaches one (e.g. `dsh-tool-fs` carries its result-time
   * contextual diff here).
   */
  'tool/result': {
    turn: number
    step: number
    message: ToolResultMessage
    error?: { name: string; code: string }
    meta?: JsonValue
  }
  /** Whole-list snapshot; latest write wins on replay. Log-only UI state; never derived history. */
  'todo/write': { todos: TodoItem[] }
  /**
   * Full header for the next request, appended inside its step before dispatch.
   * It is log-only; the latest snapshot reconstructs the request header.
   */
  'request/header': { header: EpochHeader; reason: RequestHeaderReason }
  /**
   * Route metadata for the next request, logged only when the route or capacity
   * changes. It does not participate in request reconstruction or header equality.
   */
  'request/context': RequestContext
  /**
   * Marks the end of a constructor seed. Events before it have smaller seq
   * values and came from the seed (resume, fork, or replay); this lifecycle
   * produced none of them. This log-only event is the durable projection of
   * {@link Session.firstLiveSeq}. Its payload is empty — position and `time`
   * carry the meaning.
   *
   * Locate the LAST one in stored history. A seed already ending in one is not
   * re-marked, so reopening an untouched session does not grow its log per
   * pickup and the event need not be at the current `firstLiveSeq`.
   *
   * `Session`'s constructor is the only legitimate writer. The invariant
   * companion deliberately constrains nothing here, so a plugin appending one
   * would silently classify every live bracket before it as seed history.
   *
   * An owner of a standalone open/close bracket (`compaction/start` …
   * `compaction/end`) reads it because seed history and live work are otherwise
   * byte-identical: an unmatched opening marker before this event belongs to
   * an ended lifecycle, whatever ended it. NOT a liveness signal about other
   * writers — a concurrently live session holds its own boundary elsewhere,
   * so tolerating concurrent writers needs a signal beyond the log.
   */
  'session/end-seed': Record<string, never>
}
```
`UserMessage` es el valor de rol de usuario identificado y congelado que comparten los prompts ordinarios, el contexto inyectado, el steering (direccionamiento) y los eventos de inbox en vivo. Los envoltorios de evento añaden solo hechos de posición o resultado locales al evento; el loop añade solo estado de enrutamiento propiedad del driver mientras un elemento sigue pendiente.

### `TodoItem` — una entrada de la lista de tareas

La unidad de la instantánea de lista completa del evento `todo/write`. Deliberadamente mínimo: una línea `content` y un `status` de tres estados (sin id, prioridad ni `activeForm`): la lista se reemplaza por completo en cada escritura, así que las entradas no necesitan identidad estable. Véase la [todo_write Agent Note](../../.agents/notes/implemented/feature/2026-06-29-todo-write-tool.es.md).

```ts type-equiv
/**
 * One entry in an agent's todo list — the unit of the `todo/write`
 * {@link SessionEventMap} event's whole-list snapshot.
 *
 * Deliberately minimal: a human-readable `content` line and a three-state
 * `status`. No id, priority, or `activeForm` — the list is replaced wholesale
 * on every write (last-write-wins), so entries need no stable identity. The
 * three statuses describe the complete portable lifecycle needed by model and
 * UI consumers.
 */
interface TodoItem {
  /** What this task is — a short imperative line shown in the UI. */
  content: string
  /** Lifecycle state. `in_progress` marks a task being worked now; parallel work may mark several. */
  status: 'pending' | 'in_progress' | 'completed'
}
```

<a id="the-request-header-event-requestheader"></a>

### El evento de cabecera de petición: `request/header`

El sobre de la petición — la `EpochHeader` (config de llamada + marcadores de los valores por defecto aportados por el adaptador + prompt de sistema renderizado + schemas de herramientas ensamblados) — es estado de sesión registrado en el log, de modo que cada petición de conversación es una función pura del log (la Agent Note de reconstruibilidad). Una instantánea completa de `request/header` con la razón `'initial'` o `'resume'` registra cada límite de instancia del loop; una petición posterior modificada registra otra instantánea completa con la razón `'change'`. `foldRequestHeader(events)` reconstruye la cabecera seleccionando la instantánea más reciente. El evento no es un `SurfaceEventType`: no produce ningún mensaje de LLM.

```ts type-equiv
/**
 * Logged request state outside derived history: call config, system prompt, and
 * tools. The latest full `request/header` snapshot reconstructs it; canonical
 * empty optional fields are absent.
 */
interface EpochHeader {
  /** The conversation's call configuration (provider, model, reasoning effort, and sampling scalars). */
  config: LlmCallConfig
  /** Effective config fields materialized from the exact adapter rather than proposed by a caller. */
  adapterDefaults?: LlmCallConfigAdapterDefaults
  /** Rendered system prompt text; absent for a system-less request. */
  system?: string
  /** Assembled tool schemas; absent for a tool-less request. */
  tools?: ToolSchema[]
}
```

La forma canónica representa un prompt de sistema o una lista de herramientas vacíos como un campo ausente, igual que se construyen las peticiones. Los logs v0 heredados que contengan el evento heredado `request/header-delta` o su razón `fallback` de instantánea completa se rechazan en los límites de semilla, de append y de carga de persistencia, en lugar de reproducirse de forma incompleta.

### El evento de capacidad de ruta: `request/context`

Los metadatos de contexto de la ruta a la que se resolvió una petición son estado de log independiente, que se añade junto a `request/header` dentro del mismo paso y solo cuando el provider, el modelo o la capacidad difieren del registro anterior. Queda fuera de `EpochHeader` porque ese tipo es el contrato de reconstrucción que `headerEquals` compara campo a campo: la capacidad describe una ruta, no una entrada de petición, así que plegarla en él haría que un cambio de capacidad se registrara como un `change` del sobre de petición y arrastraría los metadatos del adaptador al invariante de reconstrucción del loop. Como `request/header`, no es un `SurfaceEventType` y no produce ningún mensaje de LLM. `session.requestContext()` pliega el registro más reciente de forma incremental. Una ruta cuyo adaptador no anuncia capacidad se registra con `contextWindow` ausente, de modo que el nuevo registro limpia la capacidad de una ruta anterior.

```ts type-equiv
/** Registration-bound metadata for one resolved model route. */
interface RequestContext {
  /** Registered provider route the metadata belongs to. */
  provider: string
  /** Provider-owned model id the metadata belongs to. */
  model: string
  /** Maximum combined request and response context in tokens, when advertised. */
  contextWindow?: number
}
```
## `SessionEvent<T>` — una entrada del log

Una unión discriminada de verdad sobre `type` (no uniones independientes de `type`/`data`), de modo que `switch (event.type)` estrecha `event.data` sin casts. `seq` es la posición monótona en el log (`seq = log.length`); `time` son ms de época Unix.

```ts type-equiv
/**
 * One immutable entry in the session log.
 *
 * A proper discriminated union over `type` (not independent `type`/`data`
 * unions), so `switch (event.type)` narrows `event.data` without casts.
 *
 * The {@link sourceEventSeqs} and {@link surfaceOp} fields are conditional:
 * they only exist on {@link SurfaceEventType} variants (`user/message`,
 * `assistant/message`, `tool/result`).
 * Non-surface events (boundary markers, chunks, usage, errors) never carry
 * surface metadata — the compiler enforces this at `Session.append()`
 * call sites.
 */
type SessionEvent<T extends SessionEventType = SessionEventType> = {
  [K in SessionEventType]: {
    type: K
    /** Monotonic sequence number within the session. */
    seq: number
    /** Unix epoch milliseconds. */
    time: number
    data: SessionEventMap[K]
    /**
     * Marks an event a reader may safely skip when it does not recognize
     * `type`. Absent means required: a reader meeting an unrecognized type
     * without this marker MUST refuse to reconstruct the session instead of
     * silently dropping the event, because an unrecognized required event may
     * change how the rest of the log is interpreted. A writer sets `true` only
     * on purely informational records whose loss cannot affect reconstruction;
     * defaulting to required means a forgotten marker over-refuses (an
     * inconvenience) rather than silently resuming a gutted session.
     */
    ignorable?: true
  } & (K extends SurfaceEventType ? {
    /**
     * Seq numbers of earlier events that this event cites as sources
     * (e.g. the `assistant/chunk` seqs that built an `assistant/message`,
     * or the surface nodes shadowed by a compaction replace node). An
     * `assistant/message` may carry a present empty array for a known empty
     * provider stream; when the field is absent, the event does not record which
     * earlier events produced the message.
     */
    sourceEventSeqs?: number[]
    /** How this event entered the surface; absent for non-surface events. */
    surfaceOp?: SurfaceOp
  } : object)
}[T]
```

`SessionEventType = keyof SessionEventMap`. Como `SessionEventMap` es extensible por fusión, los switch sobre `SessionEvent` NO deben usar `assertNever`: una variante añadida por un plugin es un valor desconocido válido; gestiona los casos conocidos y cae en `default`.

Para `assistant/message`, un `sourceEventSeqs: []` presente es un stream de provider completo y conocido como vacío, mientras que un evento heredado o ajeno sin ese campo no registra qué eventos anteriores produjeron el mensaje. El loop escribe el campo en cada llamada de modelo correcta; cualquier otro evento de surface exige una lista no vacía cuando el campo está presente.

## Tipos de surface

Los tres tipos que producen mensajes (`SurfaceEventType`: `user/message`, `assistant/message`, `tool/result`) llevan metadatos de surface que declaran cómo se incorporan a la surface ordenada y derivada. Véase la [session surface Agent Note](../../.agents/notes/implemented/architecture/2026-06-18-session-surface.es.md).

### `SurfaceEventType` — el subconjunto de tipos de evento que producen mensajes

```ts type-equiv
/**
 * The subset of {@link SessionEventType} values whose events produce LLM
 * messages and are eligible to appear on the ordered surface. Only these
 * event types may carry {@link SurfaceOp} and {@link SessionEvent.sourceEventSeqs}.
 */
type SurfaceEventType =
  | 'user/message'
  | 'assistant/message'
  | 'tool/result'
```

### `SurfaceOp` — cómo entró un evento en la surface

```ts type-equiv
/**
 * How a session event entered the ordered surface. Only valid on
 * {@link SurfaceEventType} events.
 *
 * - `'append'`: added to the tail — normal path for user/assistant/tool
 *   messages.
 * - `{ op: 'replace', start, end }`: replaces surface nodes from `start`
 *   (inclusive) through `end` (inclusive) with this node. Both must exist as
 *   surface nodes in the current surface. `start === end` replaces a single
 *   node. The node's {@link SessionEvent.sourceEventSeqs} must include every
 *   shadowed surface node. Used by compaction; any surface-replacing producer
 *   may use it.
 */
type SurfaceOp =
  | 'append'
  | { op: 'replace'; start: number; end: number }
```

`'append'` es la vía normal de añadido a la cola. `replace` eclipsa las entradas de surface desde `start` hasta `end`, ambos incluidos (los dos deben ser seq de surface válidas; `start === end` reemplaza una sola entrada) e inserta el nuevo evento en su lugar.

### `SurfaceIntent` — el parámetro de `session.append()`

```ts type-equiv
/**
 * Surface placement and cited source-event seqs for {@link Session.append}. Required on
 * message-producing events and forbidden on log-only events.
 */
interface SurfaceIntent {
  surfaceOp: SurfaceOp
  /**
   * Complete set of known source-event seqs. `assistant/message` may use a
   * present empty array for a known empty provider stream; when the field is
   * absent, the event does not record which earlier events produced the message.
   * Other surface events require a non-empty set when this field is present.
   */
  sourceEventSeqs?: number[]
}
```

Obligatorio para los eventos `SurfaceEventType`: todo evento que produce mensajes debe declarar cómo se incorpora a la surface, la única fuente de la historia de modelo derivada. Un transcript (transcripción) orientado a humanos es la otra proyección y lee en su lugar los eventos de origen de append del log, porque la surface eclipsa deliberadamente los rangos que resume un reemplazo (`isAppendSurfaceEvent` en [dsh-session](../../packages/core/session/README.es.md)). Los tipos que no son de surface lo rechazan en tiempo de compilación.

Solo `assistant/message` puede llevar un `sourceEventSeqs` vacío presente; cuando el campo está ausente, el evento no registra qué eventos anteriores produjeron el mensaje, y el provider puede haber emitido fragmentos (chunks) igualmente.

### `SessionSurface` — la proyección viva de solo lectura de la surface

`Session.surface` devuelve la vista `SessionSurface` estable de la sesión. El mismo gestor incremental valida los candidatos a append antes de confirmarlos y hace avanzar esta proyección a partir de los eventos confirmados; los llamadores pueden observar la pertenencia y la generación de reemplazos, pero no pueden invocar la validación.

`SurfaceManager(log, baseSeq?)` puede en su lugar plegar una ventana contigua cargada cuyo primer evento tenga la secuencia absoluta `baseSeq`. Todos los eventos siguen siendo contiguos en ese espacio de secuencias absolutas, y un reemplazo que cruce el límite de la ventana falla porque su rango declarado no existe.

```ts type-equiv
/** Readonly live projection of the message-producing session events. */
interface SessionSurface {
  /** Current surface event sequences in model-visible order. */
  readonly nodes: readonly number[]
  /** Monotonic count of committed positional replacements. */
  readonly replaceGeneration: number
}
```

### `SurfaceFoldReplacement` y `SurfaceFoldResult` — una reproducción completa de la surface

`foldSurface(events)` devuelve las secuencias de eventos actuales desacopladas junto con las secuencias reales eclipsadas por cada rango de reemplazo declarado. El gestor en vivo usa las mismas transiciones sin retener el historial de reemplazos. Su `replaceGeneration` se incrementa con cada reemplazo confirmado, de modo que los consumidores incrementales pueden distinguir el crecimiento puro de la cola de una reescritura.

```ts type-equiv
/** One replacement operation observed while folding a session surface. */
interface SurfaceFoldReplacement {
  /** Seq of the event that replaced the prior surface range. */
  seq: number
  /** Declared inclusive start seq of the replaced surface range. */
  start: number
  /** Declared inclusive end seq of the replaced surface range. */
  end: number
  /** Actual surface entries removed by the operation, in surface order. */
  shadowedSeqs: number[]
}
```

```ts type-equiv
/** Complete result of replaying the surface operations in a session log. */
interface SurfaceFoldResult {
  /** Current surface event sequences in model-visible order. */
  nodes: number[]
  /** Replacement operations in event order. */
  replacements: SurfaceFoldReplacement[]
}
```
## API pública de `Session`

La declaración sin cuerpo mantiene sincronizados con el código fuente la fábrica desacoplada de la clase simple, sus accesores de estado, su método de append y sus proyecciones de historial. Las operaciones del store permanecen en la [sección `ctx.sessions`](#ctxsessions--sessionstore) generada.

```ts public-api
/**
 * An event-sourced session: an append-only log of {@link SessionEvent}s.
 *
 * Plain class (not a Service) — create live instances via
 * `ctx.sessions.create()` and detached instances via {@link create}.
 * Seeding with an existing event log replays/forks a session.
 * @typert object
 */
declare class Session {
  /** The ordered surface over this session's event log. */
  get surface(): SessionSurface;
  /**
   * Detached, deep-frozen creation metadata (format version, cwd, lineage,
   * seed boundary). Supplied by the store via `ctx.sessions.create()`. When a
   * `Session` is created without a store-owned header, a minimal header is
   * synthesized (stamped with the current {@link SESSION_FORMAT_VERSION}) so
   * `session.header` is always present. Kept out of the event log — it is a
   * storage concern, not replayable conversation state.
   */
  readonly header: SessionHeader;
  /** The session identity, derived from its durable header's single copy. */
  get id(): SessionId;
  /**
   * The first seq appended IN THIS PROCESS: the length of the constructor
   * seed (0 without one). Events with smaller seq values entered through
   * construction — replay, fork, or resume — and were never published on the
   * `session/event` firehose (constructor seeds do not emit), so consumers
   * that replay the log as a publication substitute (telemetry adoption)
   * start here. Distinct from `header.seedLength`, the DURABLE fork-lineage
   * boundary: a resumed session's constructor seed is its full stored log,
   * while its header keeps the original fork value — this field is the
   * in-process construction fact.
   *
   * Not persisted itself: a seeded session projects it into the log as the
   * `session/end-seed` event, which is what a consumer reading STORED history
   * reads. Locate the LAST such event, not necessarily one at this seq — a
   * seed already ending in one is not re-marked, so reopening an untouched
   * session leaves that event at a smaller seq than `firstLiveSeq`. Prefer
   * this field in-process: it is exact before the marker reaches storage.
   *
   * When this lifecycle appends the marker, it occupies this seq before the
   * store attaches and therefore does not publish either. Otherwise this seq
   * holds an ordinary published write.
   */
  readonly firstLiveSeq: number;
  /**
   * Create a detached session by validating and snapshotting borrowed seed
   * events and storage metadata.
   * @param id - session identity.
   * @param seed - optional borrowed replay or fork events.
   * @param header - optional borrowed storage metadata.
   * @returns a detached session.
   */
  static create(id: SessionId, seed?: readonly SessionEvent[], header?: SessionHeader): Session;
  /**
   * Restore a detached session by taking ownership of fresh persistence values.
   * The storage format, event envelopes, sequence continuity, surface transitions,
   * and header fields are validated before the restored objects are frozen.
   * @param id - restored session identity.
   * @param seed - fresh detached events whose ownership is transferred.
   * @param header - fresh detached metadata whose ownership is transferred.
   * @returns a restored detached session.
   */
  static fromRestore(id: SessionId, seed: readonly SessionEvent[], header: SessionHeader): Session;
  /**
   * An immutable snapshot of the append-only event log. The snapshot is reused
   * until the next append; a previously returned array does not grow later.
   * Events and their nested data are deep-frozen at acceptance, so neither a
   * cast nor ordinary JavaScript can rewrite durable history.
   */
  get events(): readonly SessionEvent[];
  /** The next event's sequence number — always the log length (the `seq = log.length` contiguity contract). */
  get seq(): number;
  /**
   * Append one typed event to the log and synchronously notify observers via
   * the store-owned, module-private publication hooks. The hot path never blocks
   * on I/O — persistence plugins buffer asynchronously. Once the event enters
   * the log, the append is committed: observer failures are logged and
   * contained per listener, so they do not change the return value or prevent
   * later listeners from observing the same accepted event.
   *
   * @param type - The event type (key of {@link SessionEventMap}).
   * @param data - The event payload; must be JSON-serializable.
   * @param opts - Surface metadata: `surfaceOp` controls how the event enters
   *   the ordered surface; `sourceEventSeqs` lists the seq numbers of earlier
   *   events this one derives from. REQUIRED for
   *   {@link SurfaceEventType} events (every message-producing event must
   *   declare how it joins the surface, the sole source of derived model
   *   history) and
   *   rejected by the compiler for non-surface types like `turn/start` or
   *   `assistant/chunk`.
   * @returns the logged event — its assigned `seq`/`time` plus the SNAPSHOT of
   *   `data` that entered the log, so reading `event.data` back sees the logged
   *   value, never the caller's still-mutable input.
   * @throws if `data` or surface metadata is not losslessly JSON-serializable
   *   (BigInt, function, symbol, undefined, negative zero, non-finite number,
   *   circular reference, sparse array, or an exotic object such as
   *   Map/Set/Date/class instance), or when the candidate violates the
   *   canonical surface contract (marker shape and eligibility, unique
   *   earlier source-event references, positional replacement validity, and complete
   *   shadowed-node coverage). One recursive pass reads, validates, and
   *   copies each nested value once, so a stateful getter cannot supply one value
   *   to validation and another to storage. The event log is the durable source
   *   of truth, so a bad event fails at the append site rather than later during
   *   a backend flush. A synchronous internal dispatch validation failure or an
   *   append reentered while this acceptance/publication boundary is open also
   *   rejects before the log changes.
   */
  append<T extends SessionEventType>(
    type: T,
    data: SessionEventMap[T],
    ...opts: T extends SurfaceEventType ? [opts: SurfaceIntent] : []
  ): SessionEvent<T>;
  /**
   * The {@link EpochHeader} in force after the log's last header event — the
   * header the NEXT request will be compared against — or undefined before
   * the first `request/header` snapshot. The live, incrementally-maintained
   * form of `foldRequestHeader(session.events)`: each header event is folded
   * once, when first seen, so a per-step read costs O(new events).
   * @returns the folded header, or undefined when no header event exists yet.
   */
  requestHeader(): EpochHeader | undefined;
  /**
   * Return the latest resolved route metadata, or `undefined` before the first
   * `request/context` event. Each event is folded once.
   * @returns the latest immutable route metadata.
   */
  requestContext(): RequestContext | undefined;
  /**
   * Derive the LLM message history by walking the ordered sequences of
   * message-producing events maintained by `surfaceOp` markers. The
   * surface is the single source of derived history: every message-producing
   * append records its `surfaceOp`, so a raw event with no marker (a chunk, a
   * turn boundary) is correctly absent, and a compaction `replace` deletes the
   * shadowed nodes from the derivation. The projection rules are
   * {@link deriveEventMessage}, folded per node.
   *
   * CACHED: each surface node is projected exactly once, when first seen — a
   * call costs O(new nodes), and a surface rewrite (a `replace`;
   * {@link SessionSurface.replaceGeneration}) rebuilds. The returned array is
   * a fresh snapshot per call (later appends never grow an array a caller
   * already holds); the `Message` objects in it are SHARED and **deep-frozen**.
   * Their content reuses the already frozen durable event data, so the cache
   * needs no second deep clone and consumers still cannot mutate the log.
   * @returns a fresh array of the shared, frozen derived history.
   */
  deriveMessages(): Message[];
  /**
   * Instance face of the pure per-node `deriveEventMessage` export from
   * `surface.ts`.
   * @param event - the event to project.
   * @returns the derived message, or null when the event produces none.
   */
  deriveEventMessage(event: SessionEvent): Message | null;
}
```
## Historia derivada: `deriveMessages()` y `deriveEventMessage()`

`Session.deriveMessages()` proyecta el log de eventos en el `Message[]` que ve el modelo: en caché (cada nodo de surface se proyecta una sola vez, cuando se ve por primera vez; una reescritura de surface lo reconstruye) y congelado (un array nuevo por llamada sobre mensajes compartidos y profundamente congelados, de modo que mutar la historia registrada a través de una proyección es irrepresentable). `deriveEventMessage(event)` es la función pura por nodo que aplica el plegado: pública para que los reconstructores externos y el invariante de desarrollo proyecten un prefijo del log con exactamente las mismas reglas y no puedan discrepar de la caché. Las reglas de proyección:

- `user/message` → un mensaje de usuario que lleva el `content` exacto; un sobre opcional sigue siendo metadatos de visualización de solo log.
- `assistant/message` → un mensaje de asistente con el provider y el modelo que lo produjeron más el estado de reproducción opcional privado del adaptador. Los eventos en bruto `assistant/chunk` son datos de reproducción/UI y se **omiten** en la derivación (el mensaje ensamblado es la autoridad). También se omite un `assistant/message` de **contenido vacío**: un paso cortado por max-tokens sin contenido registra igualmente un `assistant/message` para alojar su usage, provider y modelo, pero un turno de asistente sin contenido no debe entrar en el transcript (transcripción) del provider.
- `tool/result` → un mensaje de usuario que lleva un bloque `tool-result`.
- `user/message` (contexto inyectado, es decir, fuente distinta de `user`) → un mensaje de rol de usuario que lleva su `content` verbatim en su posición cronológica; su fuente tipada nombra al productor y lleva los datos específicos del productor.

Todo lo demás (`turn/*`, `step/*`, `llm/retry` propiedad del plugin) es estructural y no se proyecta en un mensaje. La contabilidad de tokens lee los registros `assistant/chunk { type: 'usage' }` por paso y trata `assistant/message.usage` como respaldo del paso confirmado cuando no existe un fragmento de usage; los intentos de petición al modelo fallidos no tienen mensaje de asistente, así que su fragmento de usage es el registro contable duradero. Como este formato no publicado no promete deliberadamente compatibilidad alguna, la validación de semilla/carga rechaza las cabeceras de petición y los mensajes de asistente que omiten provider/modelo en lugar de adivinar una ruta para los datos históricos.

## API de fork de sesión en vivo

`ctx.sessions.create(id, { seed, meta })` es la primitiva de bajo nivel de reproducción/fork. Para los forks ordinarios de sesiones en vivo, `SessionStore` expone una única API de política:

- `fork(source, boundary?, childSessionId?)` acepta un objeto `Session` en vivo o un `SessionId` en vivo, selecciona los eventos de la fuente hasta la seq `boundary` inclusive (por defecto: el último evento actual), exige que el prefijo seleccionado termine fuera de un turno abierto y crea una sesión hija en vivo con los eventos de semilla clonados en profundidad más los metadatos del hijo (`parentSession`, `seedLength` y el `cwd` heredado).

Un `boundary` explícito permite a los llamadores hacer fork desde cualquier posición estable entre turnos, incluido un `turn/end` anterior o un evento posterior independiente de solo log, incluso si la fuente tiene eventos más nuevos o un turno actual abierto. La API rechaza un prefijo que termine dentro de un turno abierto en lugar de recortarlo en silencio. La verificación más amplia de las relaciones de ejecución permanece en el plugin existente `dsh-invariants` y en la vía de reparación de persistencia, sin duplicarse en `fork()`. `dsh-subagent-fork-in-process` conserva su recorte de prefijos completados porque la delegación en tiempo de herramienta suele empezar mientras el turno padre está abierto; la ramificación ordinaria de sesiones debe hacer explícito el límite solicitado.

## Por qué terminó un turno: `TurnEndReasonMap`

`turn/start` no tiene campo de disparador. El lote `user/message` ingresado registra qué entró en cada paso, `llm/retry` registra la recuperación de peticiones y la inyección inactiva permanece pendiente hasta que una entrega despertadora llegue a un pre-paso posterior. Los turnos en vivo conservan el [`AgentCancelCause`](core.es.md#the-agent-handle) tipado que detuvo al driver; la persistencia usa la causa adicional `{ kind: 'legacy' }` solo al importar un registro de cancelación grueso soportado que no guardó a su llamador.

```ts type-equiv
/** Durable cancellation cause, including imports whose original coarse record carried no cause. */
type TurnEndCancelCause = AgentCancelCause | { readonly kind: 'legacy' }
```

```ts type-equiv
/**
 * Why a turn ended. Merge-extensible sum type.
 */
interface TurnEndReasonMap {
  completed: { kind: 'completed' }
  /** A cancellation request interrupted the live turn. */
  aborted: { kind: 'aborted'; reason: TurnEndCancelCause }

  blocked: { kind: 'blocked' }
  /**
   * The turn failed. `error` is always a structured failure: the `LlmError`
   * facts verbatim, or `{ message: errorChain(error), code: 'UNKNOWN' }`
   * flattened from any other error.
   */
  error: { kind: 'error'; error: LlmFailure }
  /** At least one step reached its output-token ceiling, even if a plugin continued the turn. */
  'max-tokens': { kind: 'max-tokens' }
  /**
   * A persistence backend closed a crash-orphaned turn on reload. The loop never
   * emits this marker, and the events recorded before the crash remain intact.
   */
  interrupted: { kind: 'interrupted' }
}
```

`max-tokens` refleja el `FinishReason` de llamada de modelo del mismo nombre: cualquier paso `max-tokens` en un turno hace que todo el turno termine como `max-tokens` en lugar de `completed` (el hecho de haberse cortado gana sobre una continuación posterior), de modo que un consumidor puede distinguir una parada limpia de una truncada. La cancelación y los errores siguen siendo resultados distintos. `interrupted` es la única razón que ningún loop emite: la sintetiza la recuperación ante caídas (véase [persistence.md](persistence.es.md)). El mapa es extensible por fusión.

## Delimitación de la ejecución y eventos independientes

Un turno encierra una ejecución del loop de modelo, no todo el log de la sesión. AgentLoop registra los eventos `user/message` inyectados solo a partir de los lotes de pre-paso que entran dentro de un turno; los eventos de solo log propiedad de plugins pueden aparecer igualmente entre `turn/end` y el siguiente `turn/start`, consumiendo seq de eventos sin incrementar los números de turno. La persistencia admite cada evento contiguo aceptado en un lote durable acotado, mientras que la reparación ante caídas cierra solo un turno final genuinamente abierto. Un productor que necesite una barrera de durabilidad inmediata espera explícitamente `ctx.sessions.flush(session)`.

El compañero opcional `dsh-session/invariant` hace cumplir las relaciones que pertenecen a core: la numeración de turnos y pasos, el encierro de los eventos de ejecución y el emparejamiento de llamada de herramienta/resultado en el mismo paso. Las relaciones de eventos extensibles por fusión pertenecen al plugin que las declara, así que core no rechaza un evento desconocido solo porque no haya ningún turno abierto. Véase [la decisión sobre eventos independientes](../../.agents/notes/implemented/simplification/2026-07-28-remove-synthetic-log-only-turns.es.md).

## El límite de fin de semilla: `session/end-seed`

Una sesión con semilla — reanudada, bifurcada o reproducida — añade este evento de solo log inmediatamente después de la semilla de su constructor, como su primera escritura en vivo. Los eventos anteriores tienen seq menores y proceden de la semilla. Es la proyección duradera de `firstLiveSeq`: ese campo responde dónde empiezan las escrituras de este ciclo de vida para un consumidor que tiene el objeto, mientras que el evento responde a la misma pregunta para quien solo tiene los bytes almacenados. El payload está vacío, así que la posición y el `time` cargan con todo el significado, y no produce ningún mensaje. El constructor de `Session` es el único escritor legítimo.

Una semilla vacía suministrada explícitamente escribe `session/end-seed` en la seq 0, lo que distingue una sesión reanudada vacía de una nueva. Una semilla que ya termina en `session/end-seed` no se vuelve a marcar, de modo que reabrir una sesión intacta no hace crecer su log por cada recogida. Localiza el ÚLTIMO `session/end-seed` del historial almacenado en lugar de dar por hecho que existe uno en `firstLiveSeq`: después de una recogida sin trabajo, el evento tiene una seq menor que el `firstLiveSeq` del siguiente ciclo de vida.

Existe porque el historial de la semilla y el trabajo en vivo son por lo demás byte-idénticos, lo que derrota a cualquier plugin dueño de un par de apertura/cierre independiente: un `compaction/start` sin emparejar se lee igual tanto si el escritor se estrelló a mitad de la compactación como si está compactando ahora mismo. Un marcador de apertura anterior a `session/end-seed` procede de la semilla del constructor y pertenece a un ciclo de vida terminado, lo haya terminado lo que lo haya terminado (una caída, un proceso sucesor o un fork desde un padre aún en ejecución), así que su dueño puede tratarlo como muerto. Eso cubre solo los pares que *esta* sesión heredó: una sesión concurrentemente en vivo que mantiene un par abierto sobre el mismo historial tiene su propio límite en otro sitio, de modo que tolerar escritores concurrentes necesita una señal de actividad más allá del log. Core escribe el límite y no lee nada de él: el vocabulario de un par permanece en su plugin dueño, que es por lo que la reparación ante caídas cierra los límites de turno/paso/herramienta y nunca `compaction/*`.

Los consumidores que ordenan las sesiones por actividad humana excluyen este límite: recoger una sesión no es trabajo, así que ordenar por la cola del log haría flotar hasta arriba toda sesión abierta.

## Eventos de solo log aportados por plugins

Un plugin puede fusionar declaraciones de tipos `SessionEventMap` adicionales. Estos son **de solo log**: NO son `SurfaceEventType` (no llevan `surfaceOp` y no contribuyen nada a la historia derivada). Su dueño decide si pertenecen a un turno de ejecución abierto o pueden quedar entre turnos, y hace cumplir cualquier relación en su propio compañero de invariantes. El [catálogo de eventos del log de persistencia](../persistence-catalog.es.md) generado enumera cada evento de core y aportado por plugins con su payload, su insignia de surface y su lugar de declaración; la semántica `compaction/*` del seam de compactación se trata en [compaction.md](compaction.es.md).

Cuando varios eventos de una misma familia propiedad de un plugin se ensamblan en un Conversation Node del Web Client, cada evento de inicio, actualización, resultado, recurso o interrupción de esa familia lleva o deriva de forma independiente el mismo id de negocio estable. Este requisito se aplica a las familias de Nodos correlacionadas, no a cada evento de sesión; permite al cliente agrupar cada evento sin adivinar por adyacencia ni escanear el historial. Véase el [cookbook de Conversation Node](../cookbook/adding-a-conversation-node.es.md).

Los pares `hook/invoked` / `hook/result` de los puentes de hooks (de `@deepseek-ai/dsh-hook-protocol`) se correlacionan por `handlerId`. `UserPromptSubmit`, `PreToolUse`, `PostToolUse` y `Stop` se disparan dentro del turno abierto del loop, así que sus registros `hook/*` quedan encerrados en el turno por construcción. `SessionStart` no recibe ningún registro `hook/*` porque se ejecuta antes del turno 1; su contexto permanece pendiente en el inbox hasta que una entrega despertadora abra un turno (véase la [Agent Note de puentes de hooks](../../.agents/notes/implemented/feature/2026-06-30-hook-bridges.es.md)).

## Contrato de durabilidad

En lo que confía un backend de persistencia: el log duradero persiste cada evento sin pérdidas, **incluido** `assistant/chunk`: `seq` debe permanecer contiguo, así que los fragmentos no pueden filtrarse fuera del log canónico. Un backend puede elegir su propia codificación de almacenamiento para un lote de eventos siempre que `load` devuelva exactamente los eventos añadidos (las filas de fragmentos empaquetados por defecto del backend JSONL son una codificación así: véase [persistence.md](persistence.es.md)). Todo `event.data` debe ser serializable a JSON; `Session.append` lo hace cumplir en el origen (lanzando una excepción ante datos no serializables), de modo que un evento malo nunca entra en el log y `session.events` siempre equivale a lo que un backend puede persistir. Añadir un tipo de evento que lleve datos no serializables, corrompa el anidamiento de ejecución de core o viole la relación declarada por su dueño es un cambio incompatible con el formato en disco.

Los backends que consumen este contrato están en [persistence.md](persistence.es.md).
<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxsessions--sessionstore"></a>

### `ctx.sessions` — `SessionStore`

In-memory session store (`ctx.sessions`).

Persistence is intentionally not implemented here — persistence plugins subscribe to `session/event` and flush on `session/flush` / dispose.

```ts cordis-catalog
/**
 * Create a session owned by the calling fiber: disposing that fiber stops
 * event notification and removes the session from the store. `options.seed`
 * populates the session with a copy of those events (replay/fork);
 * `options.meta` attaches creation metadata (validated absolute `cwd`, seed
 * and parent lineage, and delegation depth) as the immutable
 * {@link SessionHeader} (the store fills `version`/`id`/`createdAt`).
 *
 * For an agent whose session must be torn down IN ORDER with its loop (so the
 * loop's final events are published before the store attachment ends), do NOT use this
 * — fold the session lifecycle into the agent's own effect via
 * {@link prepare} + {@link enter} + {@link announce} (see
 * `dsh-agent-loop`'s creation transaction).
 *
 * @param id - the session id; omitted, the store mints `session-<n>`.
 * @param options - seed events and/or creation metadata for the header.
 * @returns the live session, already entered and announced.
 * @throws if a session with `id` already exists, metadata is not a plain
 *   lossless-JSON record with valid scalar fields, or `meta.cwd` is a
 *   non-absolute path (storage backends key directories off it).
 */
create(id?: SessionId, options?: CreateSessionOptions): Session

/**
 * Build a session WITHOUT entering it into the store — validate the id/cwd and
 * construct the {@link Session} (with its immutable {@link SessionHeader}).
 * Pairs with {@link enter} + {@link announce}: a caller that owns a composite
 * `ctx.effect` (the agent factory) folds the session lifecycle into that ONE
 * effect so a fiber unload tears the session + agent down as a single ORDERED
 * chain rather than as racing sibling effects — which would remove the publication hooks
 * before the driver's closing events commit, dropping them.
 *
 * @param id - the session id; omitted, the store mints `session-<n>`.
 * @param options - seed events and/or creation metadata for the header. With
 *   `seedSource: 'persistence'`, metadata and events must be fresh detached
 *   graphs whose ownership transfers to this call: they are validated and
 *   frozen in place through {@link Session.fromRestore}, so the caller must
 *   retain no mutable aliases.
 * @returns the constructed session, NOT yet in the store.
 * @throws if a session with `id` already exists, metadata is not a plain
 *   lossless-JSON record with valid scalar fields, or `meta.cwd` is a
 *   non-absolute path.
 */
prepare(id?: SessionId, options?: PrepareSessionOptions): Session

/**
 * Enter a {@link prepare}d session into the store: install the module-private
 * append publication hooks and add it to the store. Returns the DETACH
 * disposer (hooks + store removal). Does NOT emit `session/created` —
 * the caller yields this disposer inside its effect and THEN calls
 * {@link announce}, so a throwing `session/created` listener rolls the attach
 * back instead of leaking it.
 *
 * Re-checks the id for a duplicate: `prepare` and `enter` are public
 * cross-package primitives and a caller may interleave arbitrary work (or
 * another create) between them, so a stale prepared session must NOT overwrite
 * a live store entry of the same id — its detach disposer would later delete
 * the REAL session. The {@link create} convenience and the agent factory call
 * the two back-to-back so they never trip this, but the public API cannot
 * assume that.
 *
 * @param session - a {@link prepare}d session not yet in the store.
 * @returns the detach disposer (publication hooks + store removal). When called from
 *   a synchronous `session/created` listener, removal and disposal wait until
 *   that creation dispatch unwinds.
 * @throws if a session with this id is already in the store.
 */
enter(session: Session): () => void

/** Emit `session/created` exactly once for an {@link enter}ed session (with
 * the carrier {@link enter} captured). Separate from {@link enter} so the
 * caller can yield the detach disposer first (rollback safety — see
 * {@link enter}).
 * @param session - the entered session to announce to listeners.
 * @throws if the session is not live or its announcement already began,
 *   including a reentrant call from a creation listener. */
announce(session: Session): void

/**
 * Dispatch the awaited `session/flush` durability checkpoint for `session`,
 * with the carrier captured at {@link enter}. THE flush entry point: the
 * store owns the carrier, so callers (the checkpoint policy's per-request
 * barrier, goal-round-driver's idle checkpoint, teardown drains, and consumers
 * that flush themselves before reading storage) must come through here
 * rather than dispatch a raw `ctx.parallel('session/flush', …)` — one owner,
 * one spelling, and the scoped-dispatch invariant can pin it.
 * @param session - the session whose buffered events must reach durable storage.
 * @returns whether at least one durability listener participated, after every
 *   listener has settled successfully.
 * @throws the first registered listener failure after every listener settles.
 */
async flush(session: Session): Promise<boolean>

/**
 * Look up a live session.
 * @param id - the session id to look up.
 * @returns the session, or undefined when no live session has that id.
 */
get(id: SessionId): Session | undefined

/**
 * All live sessions, in creation order.
 * @returns a fresh array; mutating it does not affect the store.
 */
list(): Session[]

/**
 * Create a live child session from a stable prefix of a live source.
 * `boundary` is an inclusive source event seq; omitted means the source's
 * current last event. The selected slice may end with a between-turn event
 * but must not end inside an open turn.
 *
 * @param source - Live source session object or id.
 * @param boundary - Inclusive source event seq to fork through; omitted means
 *   the source's current last event, and omitted on an empty source forks an
 *   empty child.
 * @param childSessionId - Optional child session id; omitted delegates to
 *   `SessionStore`'s id policy.
 * @returns The created live child session.
 */
fork(source: SessionForkSource, boundary?: number, childSessionId?: SessionId): Session
```

Types: [CreateSessionOptions](persistence.es.md) · [PrepareSessionOptions](persistence.es.md) · [SessionId](core.es.md)

Source: [`packages/core/session/src/index.ts`](../../packages/core/session/src/index.ts)

<a id="session-events"></a>

### `session/*` events

<a id="sessioncreated--emit"></a>

#### `session/created` — emit

Creation announcement during session publication. A synchronous throw vetoes and rolls back with a paired disposal; detach requested during dispatch is deferred. A returned-promise rejection is logged but cannot retroactively veto this synchronous boundary. Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only sessions entered through that agent's context.

```ts cordis-catalog
/**
 * Creation announcement during session publication. A synchronous throw vetoes and rolls
 * back with a paired disposal; detach requested during dispatch is deferred.
 * A returned-promise rejection is logged but cannot retroactively veto this
 * synchronous boundary.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners
 * receive only sessions entered through that agent's context.
 * @param session - the session just entered and announced.
 * @dshScopeScan unsupported
 * @mode emit
 */
'session/created'(this: Scoped<Session>, session: Session): void
```

Types: [Scoped](scope.es.md)

Source: [`packages/core/session/src/index.ts`](../../packages/core/session/src/index.ts)

<a id="sessiondisposed--emit"></a>

#### `session/disposed` — emit

Emitted once when an announced session leaves the store, including publication rollback, but never for an entry whose creation announcement did not begin. Listener failures are logged and contained. Scope-filtered dispatch (`@deepseek-ai/dsh-scope`) reuses the owner scope.

```ts cordis-catalog
/**
 * Emitted once when an announced session leaves the store, including
 * publication rollback, but never for an entry whose creation announcement
 * did not begin. Listener failures are logged and contained.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`) reuses the owner scope.
 * @param session - the session that is no longer live in the store.
 * @dshScopeScan unsupported
 * @mode emit
 */
'session/disposed'(this: Scoped<Session>, session: Session): void
```

Types: [Scoped](scope.es.md)

Source: [`packages/core/session/src/index.ts`](../../packages/core/session/src/index.ts)

<a id="sessionevent--emit"></a>

#### `session/event` — emit

Post-commit, fire-and-forget append feed. The listener snapshot resolves before the log push, but callbacks run after it; observer failures are logged and contained without making the committed append fail. Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only events from sessions entered through that agent's context.

```ts cordis-catalog
/**
 * Post-commit, fire-and-forget append feed. The listener snapshot resolves
 * before the log push, but callbacks run after it; observer failures are
 * logged and contained without making the committed append fail.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners
 * receive only events from sessions entered through that agent's context.
 * @param session - the session whose log grew.
 * @param event - the appended event, exactly as recorded.
 * @dshScopeScan unsupported
 * @mode emit
 */
'session/event'(this: Scoped<Session>, session: Session, event: SessionEvent): void
```

Types: [Scoped](scope.es.md)

Source: [`packages/core/session/src/index.ts`](../../packages/core/session/src/index.ts)

<a id="sessionflush--parallel"></a>

#### `session/flush` — parallel

Awaited parallel durability checkpoint: every listener runs and the caller awaits all of them, with no waterfall veto. Scope-filtered dispatch (`@deepseek-ai/dsh-scope`) reuses the session's owner scope.

```ts cordis-catalog
/**
 * Awaited parallel durability checkpoint: every listener runs and the
 * caller awaits all of them, with no waterfall veto. Scope-filtered dispatch
 * (`@deepseek-ai/dsh-scope`) reuses the session's owner scope.
 * @param session - the session whose buffered events must reach durable storage.
 * @dshScopeScan unsupported
 * @mode parallel
 */
'session/flush'(this: Scoped<Session>, session: Session): Promise<void> | void
```

Types: [Scoped](scope.es.md)

Source: [`packages/core/session/src/index.ts`](../../packages/core/session/src/index.ts)
<!-- END GENERATED cordis-surface -->
