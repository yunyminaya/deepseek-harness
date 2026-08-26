# Persistencia de sesiones

[English](persistence.md) | Español

El **seam de durabilidad** del log de eventos. [session.md](session.es.md) describe el `Session` en memoria — el log `SessionEvent` de solo añadido que es la fuente de verdad. Esta página describe cómo se hace durable ese log: el servicio abstracto `SessionPersistence`, sus backends, el punto de control de flush, la recuperación ante fallos y la cabecera de metadatos que viaja junto al log. El vocabulario de eventos que transporta el log se enumera, miembro a miembro, en el [catálogo de eventos del log de persistencia](../persistence-catalog.es.md) generado.

El seam es un [seam de capacidad](../../.agents/notes/implemented/architecture/2026-06-13-capability-seams.es.md): un servicio abstracto ([dsh-session-persistence](../../packages/session/session-persistence), `ctx.sessionPersistence`) que define locate/create/append, la preparación de Session reutilizable, el load/inspect lógicos, las lecturas de sufijo físicas y la observación ligera de list/snapshot sobre el `SessionEvent` existente — **sin tipo de evento persistido paralelo** — y tres providers intercambiables que implementan el mismo contrato. Consulta el [Agent Note de session-persistence](../../.agents/notes/implemented/architecture/2026-06-14-session-persistence.es.md).

## El punto de control de flush

`session/event` es una notificación *síncrona*; los plugins de persistencia copian el evento en un controlador por sesión sin bloquear al productor. El primer evento pendiente inicia una ventana de agrupación fija, y los eventos posteriores se unen sin reiniciar su plazo. La expiración inicia un lote durable; los eventos admitidos durante esa escritura reciben su propio plazo y forman un lote posterior. `session/flush` cancela la espera y drena hasta la quiescencia, así que el loop sigue usándolo como punto de control de orden y de observación de errores antes de reclamar el siguiente turno ordinario. Una escritura en segundo plano rechazada conserva sus eventos y pausa el reintento automático; un evento nuevo inicia una ventana nueva, mientras que el flush explícito reintenta de inmediato e informa del fallo mediante `agent/error` y el logger, nunca como un evento de sesión después del turno cerrado. La eliminación realiza el mismo drenaje final. El máximo configurado acota solo la espera intencional de agrupación, no la planificación del event loop ni la latencia de durabilidad del backend ([decisión](../../.agents/notes/implemented/architecture/2026-08-08-bounded-session-persistence-write-batching.es.md)).

## La recuperación ante fallos preserva un turno interrumpido

Un backend que recarga un log que falló a mitad de turno encuentra un `turn/start` abierto sin `turn/end`. **No** trunca — un solo turno puede ser enorme en una tarea de horizonte largo (muchos pasos, salida de herramienta grande), y esos eventos se anexaron de forma durable antes del fallo. En su lugar, cierra el turno huérfano con un `turn/end { reason: { kind: 'interrupted' } }` sintético, manteniendo la ejecución interrumpida equilibrada sin cambiar ningún evento independiente anterior o posterior. `interrupted` es el único `TurnEndReason` que ningún loop emite (consulta [session.md](session.es.md#why-a-turn-ended-turnendreasonmap)).

La reparación se aplica solo a las sesiones en frío. Para un id vivo, `SessionPersistence.load(id)` espera hasta que la instantánea autoritativa en memoria sea durable y la devuelve solo cuando está equilibrada; un turno vivo abierto se rechaza en lugar de recibir fronteras de interrupción sintéticas. HMR adopta un prefijo vivo sin cerrar su turno activo.

`SessionPersistence.inspect(id)` construye una Session lógica inmutable sin publicarla ni escribir recuperación. La inspección en frío equilibra un turno interrumpido en memoria y deja intactas las colas físicas rotas; la inspección de una Session ya viva toma prestada su instantánea inmutable actual y, por tanto, puede contener un turno abierto. Las implementaciones respaldadas por un coordinador conservan la Session exacta en frío no publicada en una LRU acotada, de modo que las lecturas de historial repetidas y un `prepare(id)` posterior comparten una sola lectura, descompresión, validación, congelación y construcción de Session. `prepare(id)` reserva la Session, confirma la reparación pendiente y devuelve un manejador de publicación desechable; `load(id)` usa la misma maquinaria para confirmar la reparación sin publicación. La [decisión de preparación de Session](../../.agents/notes/implemented/architecture/2026-08-05-session-preparation.es.md) es la responsable de este ciclo de vida.

## `SessionLocation` — destino de artefacto opcional por sesión

`SessionPersistence.locate(meta)` resuelve de forma síncrona un artefacto independiente propiedad del backend sin leerlo, crearlo ni hacer flush de él. JSONL devuelve la ruta absoluta del transcript dentro de su directorio de proyecto/sesión; SQLite devuelve `undefined` porque las sesiones comparten una única base de datos. Una ruta devuelta puede, por tanto, nombrar un archivo que aún no existe o que carece del turno actual sin flush; es una pista de ubicación, no una autorización ni una garantía de frescura.

```ts type-equiv
/**
 * A backend-resolved, per-session local artifact location. The path is an
 * absolute target path and can name an artifact that has not materialized yet.
 * Consumers must treat it as a location hint, never as an authorization token.
 */
interface SessionLocation {
  /** Backend-specific artifact kind, for example `jsonl`. */
  readonly kind: string
  /** Absolute path to this session's backend-owned artifact. */
  readonly path: string
}
```

<a id="sessionheader--metadata-beside-the-log"></a>

## `SessionHeader` — metadatos junto al log

Los metadatos por sesión viajan **por separado** del log de eventos: la versión de formato, el cwd, el linaje y la frontera de seed son asuntos de almacenamiento, no eventos de conversación, así que se mantienen fuera de `SessionEventMap` y nunca llegan a `deriveMessages()`. La cabecera se adjunta a una `Session` mediante `session.header`.

Fuente: [`packages/core/session/src/types.ts`](../../packages/core/session/src/types.ts)

```ts type-equiv
/**
 * Immutable validated storage metadata, kept outside the conversation event log.
 */
interface SessionHeader {
  /**
   * On-disk format version, stamped from {@link SESSION_FORMAT_VERSION} when the
   * session is created. A persistence backend rejects any other version on load
   * (no migration — see the constant).
   */
  readonly version: number
  /** The session's id (mirrors the {@link Session}'s id). */
  readonly id: SessionId
  /** Non-negative safe-integer Unix epoch milliseconds when the session was created. */
  readonly createdAt: number
  /** Absolute working directory the session was created in (if any). */
  readonly cwd?: string
  /** The session this one was forked from (seed lineage), if any. */
  readonly parentSession?: SessionId
  /**
   * How many leading events were inherited through a seed. Persisting this
   * boundary lets resume and replay distinguish parent history from child work.
   */
  readonly seedLength?: number
  /**
   * Coarse product classification for a session created as a subagent child.
   * This is presentation metadata, not proof that the child is continuable.
   */
  readonly origin?: 'subagent'
  /**
   * Delegation depth: absent (zero) for a top-level session, parent depth + 1
   * for a subagent child. Persisted so a recursion budget survives restart and
   * resume — a runtime-only depth would reset a resumed child to top-level.
   */
  readonly delegationDepth?: number
  /**
   * Id of the agent preset this session's agent was composed from, when the
   * deployment composes per session. Durable because the preset decides the
   * session's tools and prompt: a resume that restored a different composition
   * would replay history the model can no longer act on.
   */
  readonly agentPreset?: string
}
```

## Rechazo de formato — logs que una compilación no puede leer fielmente

Un backend rechaza un log que no puede interpretar fielmente con `SessionFormatUnsupportedError`, distinto de `SessionPersistenceCorruptionError` porque no hay nada dañado. Una `version` de cabecera por delante de `SESSION_FORMAT_VERSION` nombra la dirección («escrito por un harness más nuevo — actualiza el harness para abrirlo»); una por detrás afirma que esta compilación no ofrece ninguna ruta de actualización. Tras la normalización de la forma heredada, un tipo de evento fuera del vocabulario generado de esta compilación (`KNOWN_SESSION_EVENT_TYPES`, emitido por `gen-persistence-catalog`) se rechaza del mismo modo salvo que el sobre del evento lleve `ignorable: true` — omitir en silencio un evento requerido no reconocido podría cambiar cómo debe leerse el resto del log. El mensaje añade la ruta cruda del log cuando el backend mantiene un artefacto por sesión, de modo que el texto rechazado siga siendo alcanzable. El backend JSONL rechaza una versión foránea directamente desde la línea de cabecera cruda, antes de validar la forma de la cabecera actual o de decodificar cualquier fila de evento — un formato futuro estructuralmente distinto sigue informando de la dirección de actualización, nunca de «corrupto»; SQLite valida primero la estructura del archivo completo mediante su propio pragma `SCHEMA_VERSION`. La justificación del diseño y la cadena de actualización diferida viven en el [note de session-log-version-mechanism](../../.agents/notes/implemented/architecture/2026-08-10-session-log-version-mechanism.es.md).

## `CreateSessionOptions` — seed y metadatos

Crear una `Session` a través del store toma un `seed` (historial inicial de replay o fork) y `meta` (los campos a nivel de almacenamiento que el store pliega en un `SessionHeader`). El store rellena `version`/`id` y asigna el valor por defecto de `createdAt`; el llamante puede suministrar el `cwd` absoluto validado, el linaje `parentSession`, la frontera de seed `seedLength`, el `origin` grueso opcional, el `delegationDepth`, el `agentPreset` con el que se compuso el agent y un `createdAt` existente. `origin: 'subagent'` permite que la navegación del producto oculte filas de hijos duplicadas; no prueba que un descriptor sea válido ni que el hijo pueda reanudarse.

```ts type-equiv
/**
 * Options for creating a {@link Session} via the store. `seed` replays/forks
 * an existing event log; `meta` carries the caller-supplied storage fields the
 * store folds into a {@link SessionHeader}.
 */
interface CreateSessionOptions {
  /** Initial replay or fork history supplied at construction. */
  readonly seed?: readonly SessionEvent[]
  /**
   * Storage metadata read once before publication. `seedLength` is explicit
   * because a resumed seed contains the full stored log, not only its inherited prefix.
   */
  readonly meta?: {
    readonly cwd?: string
    readonly parentSession?: SessionId
    readonly createdAt?: number
    readonly seedLength?: number
    readonly origin?: 'subagent'
    readonly delegationDepth?: number
    readonly agentPreset?: string
  }
}
```

El replay/fork es, por tanto, `ctx.sessions.create(id, { seed: seedEvents })`; reanudar una sesión *persistida* en un agent vivo es `ctx.agents.resume({ resumeSessionId })`.

## `SessionRawArtifact` — texto del artefacto almacenado verbatim

El texto de artefacto propio de un backend para una sesión, idéntico byte a byte a lo que escribió de forma durable (decodificado de su codificación física). `readRaw` lo devuelve sin reconstruirlo a partir de eventos parseados, de modo que la serialización específica del backend (empaquetado de chunks, orden de claves, saltos de línea) sobrevive. Los consumidores comprueban primero `supportsRawArtifacts`: `false` significa que el backend no ofrece esta capacidad (por ejemplo SQLite), mientras que `readRaw(...) === undefined` significa que un backend compatible no tiene ningún artefacto materializado para esa sesión.

```ts type-equiv
/** A backend's own raw artifact text for one session, verbatim. */
interface SessionRawArtifact {
  /** The session header parsed from the artifact's own first line. */
  readonly meta: SessionHeader
  /** The artifact's base filename on disk, without any physical encoding suffix. */
  readonly filename: string
  /** The artifact's full text content, decoded from the backend's physical encoding. */
  readonly content: string
}
```

## Propiedad de la preparación y la restauración

`SessionStore.prepare()` acepta opciones de creación ordinarias o grafos de persistencia nuevos transferidos mediante `RestoredSessionOptions`. La rama de restauración valida y congela en su lugar la cabecera y los eventos transferidos, de modo que los llamantes no deben conservar alias mutables. `SessionPreparation` es entonces dueña de la Session exacta no publicada hasta la publicación o el rollback; la eliminación es síncrona e idempotente. La inspección de persistencia expone solo `SessionInspection`, una vista lógica inmutable tomada prestada de la misma Session preparada.

```ts type-equiv
/**
 * Fresh storage values transferred to {@link SessionStore.prepare} without a
 * second serialization copy. Callers retain no mutable aliases.
 */
interface RestoredSessionOptions {
  /** Fresh detached storage events to validate and freeze in place. */
  readonly seed: SessionEvent[]
  /** Fresh detached storage metadata to validate and freeze in place. */
  readonly meta: SessionHeader
  /** Select the persistence ownership-transfer path. */
  readonly seedSource: 'persistence'
}
```

```ts type-equiv
/** Inputs accepted while constructing an unpublished Session. */
type PrepareSessionOptions =
  | (CreateSessionOptions & { readonly seedSource?: undefined })
  | RestoredSessionOptions
```

```ts type-equiv
/** Options for a preparation whose provider retains unpublished state. */
interface SessionPreparationOptions {
  /** Release provider-owned state when the Session was not published. */
  readonly release?: () => void
}
```

```ts public-api
/**
 * One exact unpublished Session and the provider state that keeps it usable.
 * Disposal is synchronous and idempotent. Providers decide whether release
 * returns the Session to a cache or discards it; publication may consume that
 * state before disposal, making the callback a no-op.
 */
declare class SessionPreparation implements Disposable {
  /** The exact Session to use for setup and publication. */
  readonly session: Session;
  /**
   * Wrap an unpublished Session in one preparation lifetime.
   * @param session - exact unpublished Session.
   * @param options - optional provider release behavior.
   * @returns a preparation disposed after publication or rollback.
   */
  static create(session: Session, options?: SessionPreparationOptions): SessionPreparation;
  /** Release provider state once when this preparation leaves its caller. */
  [Symbol.dispose](): void;
}
```

```ts type-equiv
/** Immutable logical session prepared from persistence or a live owner. */
interface SessionInspection {
  /** Validated immutable session metadata. */
  readonly meta: SessionHeader
  /** Validated contiguous logical event log. */
  readonly events: readonly SessionEvent[]
}
```

## Revisiones ligeras de la fuente

Los consumidores de estado derivado comparan una revisión opaca barata antes de cargar un log de eventos completo. El backend de persistencia es dueño de su representación y la cambia de forma transaccional con el append o la reparación de load que muta; los llamantes la comparan solo por igualdad.

```ts type-equiv
/**
 * Backend-owned token that identifies both one storage source and one revision
 * of a persisted session log.
 */
type SessionPersistenceRevision = Branded<'SessionPersistenceRevision'>
```

```ts type-equiv
/** Lightweight immutable source identity returned without loading a full log. */
interface SessionPersistenceSnapshot {
  /** Detached metadata for one materialized session. */
  header: SessionHeader
  /** Opaque source-qualified token that changes whenever this stored log changes. */
  revision: SessionPersistenceRevision
}
```

## Los backends

Todos implementan la misma `SessionPersistence` abstracta (locate/create/append/prepare/load/inspect/readFrom/list/listSnapshots sobre `SessionEvent`, con cancelación opcional en los métodos de observación) y pasan la suite compartida `runPersistenceContract`:

- **[dsh-session-persistence-jsonl](../../packages/session/session-persistence-jsonl)** — un log JSONL lógico de solo añadido por sesión, almacenado por defecto como tramas Zstandard concatenadas con checksum o como líneas crudas según configuración, con escrituras atómicas a prueba de fallos, recuperación de turnos interrumpidos y una ruta de read/replay.
- **[dsh-session-persistence-sqlite](../../packages/session/session-persistence-sqlite)** — un backend `node:sqlite` opcional que usa el schema 17 para almacenar ejecuciones delta exactas del mismo bloque en filas físicas acotadas `text-chunks`, `reasoning-chunks` y `tool-call-chunks`. Reconstruye el flujo de eventos lógico completo antes de devolverlo, empaqueta solo los lotes recién durables y rechaza los schemas más antiguos en lugar de migrarlos.

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.es.md).

<a id="ctxsessionpersistence--sessionpersistence-abstract-seam"></a>

### `ctx.sessionPersistence` — `SessionPersistence` (abstract seam)

Durable append-only session storage. Implementations preserve contiguous, losslessly JSON-serializable events; append resolves only after durability, and load balances a complete interrupted tail without rewriting committed events.

```ts cordis-catalog
/**
 * Resolve this backend's independent local artifact for a session without
 * reading, creating, flushing, or otherwise materializing it. Backends such
 * as SQLite that do not own one artifact per session return `undefined`.
 * @param meta - the immutable session header whose artifact is requested.
 * @returns the backend-specific absolute location, when one exists.
 */
abstract locate(meta: SessionHeader): SessionLocation | undefined

/**
 * Read a session's backend-owned artifact text verbatim — the exact durable
 * bytes the backend wrote (decoded from its physical encoding, e.g. a
 * decompressed JSONL). The returned `content` is the raw text, not a
 * reconstruction from parsed events, so it preserves backend-specific
 * serialization (chunk packing, key order, line breaks). Callers first test
 * {@link supportsRawArtifacts}; `undefined` then means only that the requested
 * session has no materialized artifact.
 * @param _id - the persisted session to read (unused by the default: no
 * per-session artifact).
 * @param signal - optional cancellation for backend read work.
 * @returns the raw artifact plus its parsed header, or `undefined` when the
 * session is absent.
 * @throws when this backend does not expose per-session raw artifacts.
 */
readRaw(_id: SessionId, signal?: AbortSignal): Promise<SessionRawArtifact | undefined>

/**
 * Register a new session's metadata. A backend MAY defer the physical write
 * until the first {@link append} (lazy materialization), in which case a
 * created-but-never-appended session is absent from {@link list}
 * — abandoned sessions leave nothing behind.
 * @param meta - the immutable header (id, version, cwd, lineage) to record.
 */
abstract create(meta: SessionHeader): Promise<void>

/**
 * Durably persist a batch of events. Honors the append-only and contiguous-
 * seq contracts: the first event's `seq` MUST equal the stored next-seq
 * (after `load` has durably closed any interrupted turn). Rejects non-JSON-
 * serializable `event.data` with an error naming the offending event type.
 * @param id - the session the batch belongs to.
 * @param events - the contiguous batch to persist, in seq order.
 */
abstract append(id: SessionId, events: readonly SessionEvent[]): Promise<void>

/**
 * Prepare the exact unpublished Session used by resume. Implementations may
 * reuse object graphs retained by an earlier {@link inspect} after confirming
 * their durable revision is still current; disposal releases an unpublished
 * reservation. Revision retries require the durable log to remain unchanged
 * for one read/check round trip; continuous external writers may delay completion.
 * @param id - persisted session to prepare.
 * @param signal - optional cancellation for preparation work.
 * @returns one owned unpublished Session preparation.
 */
async prepare(id: SessionId, signal?: AbortSignal): Promise<SessionPreparation>

/**
 * Load an immutable balanced logical view and commit any required cold
 * recovery. A complete interrupted final turn is preserved and durably
 * closed with missing tool errors plus any open step and turn boundaries;
 * only a torn final record is discarded. Unknown versions and corruption in
 * the committed prefix reject. Implementations MUST NOT crash-repair an
 * identity still bound to a live Session: a balanced live log may return as a
 * durable snapshot, while an open live turn rejects. Returned values may be
 * shared with immutable live or prepared state and must not be mutated.
 * Revision-based implementations may wait for one stable read/check round trip.
 * @param id - the persisted session to reload.
 * @returns the header and a log ending on a balanced `turn/end`.
 */
abstract load(id: SessionId): Promise<SessionInspection>

/**
 * Inspect an immutable logical session without committing recovery or
 * publishing it. A cold complete interrupted turn receives synthetic closers
 * in memory and a torn physical tail remains untouched. An already-live
 * Session instead yields its current immutable snapshot, which may contain an
 * open turn and its `session/end-seed` boundary. Coordinator-backed
 * implementations retain the exact cold unpublished Session for bounded
 * reuse by a later {@link prepare}. A stale ready source is reloaded; a source
 * already committing or reserved for resume remains exclusive, and inspection
 * may borrow its immutable view. Callers borrow only the immutable header and
 * log. Continuous external writers may delay revision convergence.
 * @param id - the persisted session to inspect.
 * @param signal - optional cancellation for queued and backend read work.
 * @returns the validated header and current logical event log.
 */
abstract inspect(id: SessionId, signal?: AbortSignal): Promise<SessionInspection>

/**
 * Read the stored events from `fromSeq` onward — the read-from-seq
 * primitive for read models that resume from a watermark (e.g. a persisted
 * projection cache folding only the tail past its checkpoint). Unlike
 * {@link inspect}, it is a detached physical suffix read: no preparation
 * cache, torn-tail truncation, synthetic closers, or coordinator-state
 * publication. Only events from the valid contiguous stored prefix are
 * returned, so a torn fragment never reaches the caller. `fromSeq` at or
 * beyond the stored prefix returns an empty event list (never an error).
 * Backends whose medium can seek by seq
 * (SQLite) read only the suffix; sequential media (JSONL, both encodings)
 * still parse the whole artifact and skip forward — the primitive bounds
 * what is RETURNED and refolded, not every backend's physical read.
 * @param id - the persisted session to read.
 * @param fromSeq - first event seq to include; a non-negative safe integer.
 * @param signal - optional cancellation for queued and backend read work.
 * @returns the header and the stored events with `seq >= fromSeq`.
 */
abstract readFrom(id: SessionId, fromSeq: number, signal?: AbortSignal): Promise<{ meta: SessionHeader; events: SessionEvent[] }>

/**
 * Lightweight listing from metadata, without a full-log parse.
 * @param signal - optional cancellation for backend listing work.
 * @returns one header per materialized session.
 */
abstract list(signal?: AbortSignal): Promise<SessionHeader[]>

/**
 * List materialized sessions with cheap per-log change tokens.
 *
 * Repeated observations of an unchanged log return the same revision. A
 * successful mutating {@link load} repair changes the next listed revision.
 * Revisions also distinguish independently backed stores so backend-local
 * counters cannot compare equal across different persistence sources.
 * @param signal - optional cancellation for backend snapshot-listing work.
 * @returns one header and opaque revision per materialized session without loading full logs.
 */
abstract listSnapshots(signal?: AbortSignal): Promise<SessionPersistenceSnapshot[]>
```

Types: [SessionEvent](session.es.md) · [SessionId](core.es.md)

Source: [`packages/session/session-persistence/src/index.ts`](../../packages/session/session-persistence/src/index.ts)
<!-- END GENERATED cordis-surface -->
