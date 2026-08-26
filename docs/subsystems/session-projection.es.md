# Proyecciones de sesión

[English](session-projection.md) | Español

El seam de proyecciones de sesión — un [seam de capacidad](../capability-seams.es.md) a través del cual los plugins de dominio del host sirven valores actuales completos del estado por sesión derivado del log a los portadores de cliente: la Service Definition y el registro ([dsh-session-projection](../../packages/session/session-projection), `ctx.sessionProjections`), los contribuyentes de dominio (cada uno registra una unidad pura) y los portadores (la página de cola de historial de [dsh-host-apiproxy](../../packages/host/apiproxy) y el frame de push `session/projection`). Es una capacidad opcional, no parte de la columna del agent-loop. El framework impulsa, el dominio calcula: el registro se suscribe una vez a `session/event` y hace fold de cada evento confirmado a través de cada unidad; los dominios no tienen suscripciones y los clientes nunca hacen fold de eventos de dominio — reciben valores terminados. Autoridad de diseño: la [RFC de proyecciones de sesión](../../.agents/notes/proposed/architecture/2026-07-27-session-projection-and-command-log.es.md); contratos de impulso/caché/feed: el [README del paquete](../../packages/session/session-projection/README.es.md).

Source: [`packages/session/session-projection/src/index.ts`](../../packages/session/session-projection/src/index.ts)

## La unidad

`SessionProjectionStateMap` es la tabla fusionable por extensión de estados de fold del host, mientras que `SessionProjectionMap` conserva los valores completos visibles para el cliente. Un dominio contribuye una `ProjectionDefinition` por clave de estado; un bloque `wire` hace visible esa clave para el cliente, y el renderizado pertenece al sistema de slots, nunca a esta capa:

```ts type-equiv
/**
 * One domain's state-driven computation unit: a pure synchronous fold plus
 * declarations and an optional client view — never an opaque getter. The framework drives
 * `apply` on every committed session event; the domain holds no
 * subscriptions and owns only the computation. All functions MUST be
 * synchronous (an async unit would tear the carriers' consistency cut), and
 * `state` MUST be plain JSON (the persisted-cache precondition).
 */
interface ProjectionDefinition<
  K extends keyof SessionProjectionStateMap,
  S extends SessionProjectionStateMap[K] = SessionProjectionStateMap[K],
> {
  /** The projection key this unit owns (its `SessionProjectionStateMap` entry). */
  key: K
  /** Validates persisted state before it seeds a fold. */
  stateSchema: ZodType<S>
  /**
   * State for the empty log.
   * @returns the initial state.
   */
  init(): NoInfer<S>
  /**
   * Pure transition: previous state + one committed event → next state. A
   * unit uninterested in an event MUST return the same state reference — an
   * unchanged reference (`Object.is`) produces zero downstream work.
   * @param state - the state covering all prior events.
   * @param event - the next committed session event.
   * @returns the next state (same reference when the event is not the unit's).
   */
  apply(state: NoInfer<S>, event: SessionEvent): NoInfer<S>
  /** Client view. Omit for host-only units. */
  wire?: K extends keyof SessionProjectionMap ? {
    /** Validates the wire payload before it leaves the host. */
    viewSchema: ZodType<SessionProjectionMap[K]>
    /**
     * State → wire payload (the read-side projection).
     * @param state - the current state.
     * @returns the whole current value for this unit's key.
     */
    view(state: NoInfer<S>): SessionProjectionMap[K]
  } : never
  /**
   * Persisted-cache invalidation version: bump whenever the serialized state fields or the
   * fold semantics change, so persisted `(sessionId, key, ver, seq, val)`
   * rows from an older unit are discarded instead of being forward-applied
   * into garbage. Non-negative integer.
   */
  stateVersion: number
}
```

La regla del evento de valor completo es estructural: un evento de log que porta estado transporta el estado completo posterior al cambio, nunca un delta pelado — mantiene cada transición trivialmente barata y cada valor servido autodescriptivo (gana el último para los Consumer).

## La instantánea y el feed de cambios

```ts type-equiv
/**
 * One consistent read cut over every registered client-visible unit for one session.
 * `asOfSeq` is the shared watermark — the seq of the last event every value
 * reflects (`-1` for an empty log, mirroring `session/subscribed.lastSeq`).
 */
interface ProjectionSnapshot {
  /** Seq of the last event the values reflect; -1 for an empty log. */
  asOfSeq: number
  /** Whole current client value per registered key. */
  values: Partial<SessionProjectionMap>
}
```

```ts type-equiv
/**
 * Change-feed listener: one unit's value changed for one session. `value` is
 * the schema-validated `view` output; `seq` is the unit's watermark at
 * emission (the seq of the event that caused the change).
 */
type ProjectionChangeListener = (
  session: Session,
  key: Extract<keyof SessionProjectionMap, string>,
  value: unknown,
  seq: number,
) => void
```

`snapshot(session)` es totalmente síncrono: un portador la lee en el mismo tick que su rebanada de página, así que `asOfSeq` cubre ambas lecturas con un único número de secuencia. Devuelve solo vistas de cliente, y cada valor pasa por el `viewSchema` de su unidad antes de devolverse. `stateOf(session, key)` lee un estado del host en vivo sin calcular vistas no relacionadas; quienes llaman no deben mutar la referencia prestada. El feed de cambios se dispara una vez por unidad visible para el cliente cuya *referencia* de estado cambió por cada evento confirmado; `apply` debe devolver la misma referencia cuando su estado no cambió.

## El registro: `ctx.sessionProjections`

`SessionProjectionRegistry` ([firmas](#ctxsessionprojections--sessionprojectionregistry)) es el dueño del impulso: una suscripción a `session/event`, `apply` ansioso sobre cada unidad registrada y celdas de watermark por sesión y por unidad. Las celdas se construyen de forma perezosa — una unidad registrada después de que fluyeran eventos, o una sesión más antigua que el registro, hace fold de `init` sobre el log en memoria en el primer toque (evento o lectura). El registro es un efecto cuyo disposer viaja en la fibra que llama: la clave de un plugin de dominio descargado (con sus celdas en caché) desaparece de los impulsos e instantáneas posteriores, y los clientes lo leen como ausencia de capacidad; las claves duplicadas lanzan error. Los plugins de dominio se registran bajo `ctx.inject(['sessionProjections'], …)` para que los ensamblajes headless sin el registro no se vean afectados.

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## API de Cordis

Generado desde la fuente por `scripts/gen-cordis-catalog.ts` (verificado como fresco por `pnpm run verify-cordis-catalog` en doc-sync; regenéralo con `pnpm run gen-cordis-catalog`) — los lados de idioma solo difieren en las rutas de los documentos emparejados específicas de cada localización. Los bloques de firmas usan un recinto `ts cordis-catalog` y conservan el JSDoc original de la fuente; los modos de despacho están definidos en el [primer](../cordis-primer.es.md#dispatch-modes), y la API `ctx` heredada del framework vive en [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxsessionprojectioncache--sessionprojectioncache"></a>

### `ctx.sessionProjectionCache` — `SessionProjectionCache`

El servicio de caché de proyección persistido. Abre el dominio `session_projcache` en init, hace checkpoint de las sesiones en vivo con una escritura diferida limitada (disparadores de recuento/intervalo de Config) más dos puntos obligatorios — `turn/end` y la disposición de la sesión (el momento de en vivo a frío) — y sirve la escalera de lectura en frío: fila en caché, cola `readFrom` de persistencia, `restore` del registro, escritura diferida duradera. Cada escritura duradera es de fallo blando: los fallos registran una advertencia y el caché se autocura en la siguiente escritura o lectura en frío.

```ts cordis-catalog
/**
 * The zero-I/O listing read: whole values viewed straight from the stored
 * rows (version-matching keys only), each cut carried with its watermark
 * so a client value store can seed under its higher-seq-wins rule — as
 * stale as the last durable checkpoint but never wrong, and never from an
 * unrelated log (the caller's header is the identity witness). Fresher
 * paths (the history tail baseline, {@link coldSnapshot}) supersede these
 * values whenever a session is actually opened.
 * @param meta - the listed session's header (identity witness; no log read).
 * @returns the cut (`asOfSeq` = lowest served-row watermark), or
 *   `undefined` when no usable row exists for this lifecycle.
 */
cachedSnapshot(meta: SessionHeader): ProjectionSnapshot | undefined

/**
 * Durably checkpoint one live session NOW (both mandatory points call
 * this; tests and carriers may too). The registry cut is snapshotted at
 * this boundary (states are live references), then the whole record is
 * replaced. NOT fail-soft — callers on the fail-soft paths contain it.
 * @param session - the live session to checkpoint.
 * @returns resolution after durability and event emission.
 */
async write(session: Session): Promise<void>

/**
 * Cold-read one persisted session's projections with zero full-log load:
 * cached rows + a persistence `readFrom` tail from the registry's restore
 * floor, refolded by the registry and written back (fail-soft) so the next
 * cold read starts closer. A cache row invalidated by a shrunk log
 * (crash-repair truncation) triggers one full re-read from seq 0 — the
 * ladder's slow rung, still no crash. Rejects when the session has no
 * persisted log (`not found` from the persistence seam).
 * @param id - the persisted session to read.
 * @param signal - optional cancellation for the persistence reads.
 * @returns the snapshot cut at the stored log end.
 */
async coldSnapshot(id: SessionId, signal?: AbortSignal): Promise<ProjectionSnapshot>
```

Types: [Session](session.es.md) · [SessionHeader](persistence.es.md) · [SessionId](core.es.md)

Source: [`packages/session/session-projection-cache/src/index.ts`](../../packages/session/session-projection-cache/src/index.ts)

<a id="ctxsessionprojections--sessionprojectionregistry"></a>

### `ctx.sessionProjections` — `SessionProjectionRegistry`

`ctx.sessionProjections`: la tabla de unidades de proyección y su impulso. El servicio se suscribe una vez a `session/event`; cada evento confirmado pasa por el `apply` de cada unidad registrada (impulso ansioso), y una referencia de estado cambiada en una unidad visible para el cliente notifica al feed de cambios con la vista validada por schema. Las celdas se construyen de forma perezosa — una unidad registrada después de que fluyeran eventos, o una sesión más antigua que el registro, hace fold de `init` sobre el log en memoria en el primer toque (evento o lectura). El registro es un efecto (el disposer viaja en la fibra que llama): la clave de un plugin de dominio descargado desaparece de las instantáneas y los clientes lo leen como ausencia de capacidad. Los plugins de dominio se registran bajo `ctx.inject(['sessionProjections'], …)` para que los ensamblajes headless sin el registro no se vean afectados. Los registrantes que comparten una clave comparten una unidad y se cuentan: el mismo paquete de herramienta montado en N presets de agent se registra N veces, y la clave sobrevive hasta que se descarga el último.

```ts cordis-catalog
/**
 * Register one domain's unit. The registration is an effect on the calling
 * context's fiber: disposing the fiber (or calling the returned disposer)
 * removes the key — and the unit's cached cells — from subsequent drives
 * and snapshots.
 * @param definition - key, state schema, pure unit functions, and stateVersion.
 * @returns the exact disposer that unregisters this unit.
 */
register< K extends keyof SessionProjectionMap, S extends SessionProjectionStateMap[K], >( definition: Omit<ProjectionDefinition<K, S>, 'wire'> & { wire: NonNullable<ProjectionDefinition<K, S>['wire']> }, ): () => void

/**
 * Register one host-only unit. Its state is omitted from client snapshots
 * and always checkpointed like every other unit.
 * @param definition - key, state schema, pure unit functions, and stateVersion.
 * @returns the exact disposer that unregisters this unit.
 */
register< K extends Exclude<keyof SessionProjectionStateMap, keyof SessionProjectionMap>, S extends SessionProjectionStateMap[K], >( definition: Omit<ProjectionDefinition<K, S>, 'wire'>, ): () => void

/**
 * Subscribe to the change feed. The registration is an effect on the
 * calling context's fiber.
 * @param listener - called once per client-visible unit whose state reference changed, per committed event.
 * @returns the exact disposer that unsubscribes.
 */
onChanged(listener: ProjectionChangeListener): () => void

/**
 * Read one unit's current host state without computing unrelated views.
 * The returned value is live; callers must not mutate it.
 * @param session - the session whose state is read.
 * @param key - the registered unit key.
 * @returns current state, or `undefined` when the key is not registered.
 */
stateOf<K extends keyof SessionProjectionStateMap>( session: Session, key: K, ): SessionProjectionStateMap[K] | undefined

/**
 * One consistent cut over every registered client-visible unit for one session, read from
 * the watermark cache (missing cells fold lazily over the in-memory log).
 * Fully synchronous — every value and `asOfSeq` reflect the same log
 * position. Each value passes its unit's `viewSchema` before leaving.
 * @param session - the session whose projection values are read.
 * @returns the snapshot; `values` is empty when no client-visible unit is registered.
 */
snapshot(session: Session): ProjectionSnapshot

/**
 * State-level checkpoint of every persisted unit for one session, read
 * from the watermark cache (missing cells fold lazily over the in-memory
 * log). This is the write side of the persisted projection cache: the
 * returned rows are the `(key → {ver, seq, val})` part of the durable
 * `(sessionId, key, ver, seq, val)`
 * rows. Every `val` is a DETACHED structured clone — never the live
 * cell reference: the watermark cache is this registry's authoritative
 * mutable state, and a caller reaching the live reference could corrupt
 * every subsequent snapshot and frame through it (plain JSON by the unit
 * contract, so the clone is total).
 * @param session - the session whose unit states are checkpointed.
 * @returns one row per registered key.
 */
checkpoint(session: Session): ProjectionCheckpoint

/**
 * The stored seq a {@link restore} tail read over `checkpoint` must start
 * at: one event BELOW the lowest usable watermark (a row is usable when
 * its `ver` matches the live unit's `stateVersion`; an absent or mismatched row
 * pulls the floor to `0` — that key must refold the full log). The
 * one-below anchor is load-bearing: the tail then proves how far the
 * stored log still extends, so {@link restore} can detect a log that
 * shrank below a row's watermark (crash-repair truncation) instead of
 * serving the stale row as current — an empty tail read from the anchor
 * yields an end below every watermark and the restore rejects for a full
 * re-read.
 * @param checkpoint - persisted rows for one session (possibly stale or empty).
 * @returns the seq to hand the persistence `readFrom`, or `undefined`
 *   when no unit is registered (no read needed — {@link restore} would
 *   serve empty values regardless).
 */
restoreFloor(checkpoint: ProjectionCheckpoint): number | undefined

/**
 * View a checkpoint's rows without any log read: for every registered
 * client-visible unit whose row's `ver` matches, serve the schema-validated
 * `view` of the schema-validated stored state; mismatched, malformed, or absent rows leave their key
 * absent (a cold or listing consumer treats it as not-yet-available and a
 * fuller read path refolds it). The zero-I/O rung of the read ladder —
 * values are as stale as their rows, never wrong.
 * @param checkpoint - persisted rows for one session (possibly stale or empty).
 * @returns whole values per key with a usable row; empty when none.
 */
viewCheckpoint(checkpoint: ProjectionCheckpoint): Partial<SessionProjectionMap>

/**
 * Cold read: fold every persisted unit over a stored log suffix, seeding
 * each from its checkpoint row when usable — the one read recipe (cached
 * state + forward tail replay + `view`) applied without a live `Session`.
 * Call with the events returned by a persistence
 * `readFrom(id, restoreFloor(checkpoint))` and that same floor as
 * `baseSeq`; the floor's one-below anchor makes the supplied end honest,
 * so a shrunk log is detected here. A row is usable iff its
 * `ver` matches the live unit's `stateVersion`, it does not predate `baseSeq`
 * (`seq >= baseSeq - 1`), and it does not claim events past the
 * supplied end (`seq <= endSeq`); an unusable row is discarded
 * and its key refolds from `init` — which is only sound over the full
 * log, so a discarded row with `baseSeq > 0` throws (the caller re-reads
 * from seq 0, e.g. after a crash-repair truncation shrank the log below
 * a row's watermark).
 * @param checkpoint - persisted rows for one session (possibly stale or empty).
 * @param events - the stored events with `seq >= baseSeq`, in seq order.
 * @param baseSeq - the seq `events` starts at (its first event's seq when non-empty).
 * @returns the snapshot cut at the supplied log end (`asOfSeq` is the last
 *   supplied event's seq, `baseSeq - 1` for an empty tail) plus the
 *   refreshed checkpoint rows at that cut, ready for a durable write-back.
 */
restore( checkpoint: ProjectionCheckpoint, events: readonly SessionEvent[], baseSeq: number, ): { snapshot: ProjectionSnapshot; checkpoint: ProjectionCheckpoint }
```

Types: [Session](session.es.md) · [SessionEvent](session.es.md)

Source: [`packages/session/session-projection/src/index.ts`](../../packages/session/session-projection/src/index.ts)
<!-- END GENERATED cordis-surface -->
