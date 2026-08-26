# Telemetría de sesión

[English](session-telemetry.md) | Español

El reporte de sesión saliente se divide como un [seam de capacidad](../capability-seams.es.md): la Service Definition y el coordinador de captura ([dsh-session-telemetry](../../packages/session/session-telemetry), `ctx.sessionTelemetry`) son titulares de los puntos de captura, la proyección en fragmentos fijos, el waterfall (cascada de eventos) de expurgo de `session-telemetry/record`, el cursor de handoff y el contrato mínimo de backend; el Service Provider que carga un despliegue ([dsh-session-telemetry-otel](../../packages/session/session-telemetry-otel)) es la canalización de logs del SDK de OpenTelemetry JS configurada tal cual. Es una capacidad opcional, ajena a la columna vertebral del agent loop (bucle del agente), y nada de esto llega a una petición de modelo. El axioma de frontera —la responsabilidad del harness termina en `emit()`; el procesamiento por lotes, los reintentos, el encolado y la política de pérdidas pertenecen al SDK de reporte— y las alternativas descartadas quedan fijados en el [Agent Note de reactivación](../../.agents/notes/implemented/feature/2026-07-23-session-telemetry-otel-revival.es.md); los contratos de puntos de captura, cursor y proyección viven en el [README de la Service Definition](../../packages/session/session-telemetry/README.es.md).

Fuente: [`packages/session/session-telemetry/src/index.ts`](../../packages/session/session-telemetry/src/index.ts)

## El registro lógico

```ts type-equiv
/**
 * Severity of a telemetry record, pre-mapped at capture so a receiver can
 * alert with zero configuration: `error` for events whose own outcome flag
 * says so (the tool-result block's `isError`, `turn/end` error reasons) and for
 * `agent-error` operational records. Captured events otherwise default to
 * `info`; `warn` remains available to `session-telemetry/record` policies and
 * backends.
 */
type SessionTelemetrySeverity = 'info' | 'warn' | 'error'
```

```ts type-equiv
/**
 * One logical record handed to a backend — the capture contract's whole outbound
 * vocabulary. Ledger records mirror session-log events one-to-one;
 * operational records (`channel: 'ops'`) carry the two signals with no log
 * home (`agent-error`, `shutdown`) and deliberately omit `event.seq`-style
 * identity so they can never be mistaken for ledger rows.
 */
interface SessionTelemetryRecord {
  /** Ledger (session-log mirror) or ops (operational signal) channel; backends keep the two under separate instrumentation scopes. */
  channel: 'ledger' | 'ops'
  /** Unix epoch milliseconds — the source event's append time for ledger records, the emission time for ops records. */
  time: number
  /** Pre-mapped alerting severity; see {@link SessionTelemetrySeverity}. */
  severity: SessionTelemetrySeverity
  /**
   * Identity attributes, deliberately minimal: ledger records carry
   * `session.id`, `event.type`, `event.seq`, plus `session.cwd` /
   * `session.parent_id` / `session.seed_length` when the header has them;
   * ops records carry `telemetry.op`, `session.id`, and (for `agent-error`)
   * `agent.id`, `turn`, `step`, `error.name`. Anything recoverable from the
   * body is intentionally NOT duplicated here.
   */
  attributes: Record<string, string | number>
  /**
   * The complete payload: a deep copy of the session event's `data` for
   * ledger records (JSON-serializable by `Session.append`'s own
   * validation), or the op payload for ops records. Never mutated after
   * handoff.
   */
  body: unknown
}
```

Solo se envía el primer `assistant/chunk` de cada `(turn, step)` —la señal de que el stream ha comenzado—; el resto se descarta en la captura, de modo que los huecos en `seq` son habituales en el cable y nunca indican una pérdida. Todos los demás tipos de [evento de sesión](session.es.md), incluidos los fusionados por plugins de los que el seam nunca tuvo noticia, pasan intactos. La entrega es de mejor esfuerzo: el cursor marca el traspaso, no la entrega; los registros pueden perderse (caídas, ventana de recarga) y duplicarse (readopción sin cursor, reintentos del SDK), así que los receptores deduplican los registros de ledger por `(session.id, event.seq)`; los registros ops omiten deliberadamente esa identidad —son señales para alertar, no entradas que sumar— y toleran los duplicados.

## La divulgación de uso compartido

El contrato de reconocimiento del seam (que pertenece a la [sección de divulgación de uso compartido del README de la Service Definition](../../packages/session/session-telemetry/README.es.md#the-sharing-disclosure)): cada backend divulga su política de uso compartido elegida en el despliegue a través del miembro abstracto obligatorio `sharing` de `ctx.sessionTelemetry`, y los consumidores muestran «not configured» solo cuando no hay ningún servicio de telemetría montado. La divulgación expone la política vigente, nunca la entrega ni la retención —el traspaso es el encolado no bloqueante, y el procesamiento por lotes, los reintentos y la política de pérdidas siguen siendo del SDK de reporte—.

```ts type-equiv
/**
 * Deployment-selected session-sharing policy disclosed by a mounted
 * {@link SessionTelemetryBackend} backend to human-facing acknowledgement surfaces (the
 * `/feedback` command's confirmation text). The seam owns the vocabulary so
 * any backend can disclose a policy without depending on the OTel package;
 * the values mirror the OTel backend's serialized `SessionTelemetryMode` choices.
 */
type SessionTelemetrySharingStatus = 'full' | 'feedback-only' | 'disabled'
```

## El contrato del backend

```ts type-equiv
/**
 * The minimum backend contract the coordinator requires. {@link SessionTelemetryBackend} is
 * its service-registered form; tests compose the coordinator with a bare
 * implementation of this interface.
 */
interface SessionTelemetrySink {
  /**
   * Hand one record to the backend's pipeline. MUST be a non-blocking
   * enqueue — the coordinator calls this synchronously from the
   * `session/event` hot path or an explicit canonical-log capture, so anything
   * slower than a queue push would tax the agent loop or feedback handling.
   * Errors thrown here are contained by the coordinator and logged; they
   * never reach the loop.
   * @param record - the logical record to report; owned by the backend after the call.
   */
  emit(record: SessionTelemetryRecord): void
  /**
   * Optional hint that a turn ended. A backend may forward it to its SDK's
   * flush so records are exported after each turn. Called
   * fire-and-forget; implementations must not block and must not throw
   * meaningfully (the coordinator contains exceptions). Most backends should
   * leave this unimplemented and let their SDK's own batching cadence govern
   * export timing: a backend that does implement it owns the interaction
   * between its concurrent flushes and {@link shutdown}'s drain (the OTel
   * backend leaves it unimplemented for exactly that hazard — see the
   * revival Agent Note).
   */
  flush?(): void
  /**
   * Forward the fiber's disposal to the SDK: flush whatever is queued and
   * reach quiescence, per the SDK's own shutdown contract. Everything
   * emitted before this call must still be delivered — including records
   * enqueued while a {@link flush} hint is in flight, so a backend whose SDK
   * guards against concurrent flushes orders behind the outstanding one (the
   * coordinator emits its dispose-time `shutdown` markers immediately before
   * calling this). Awaited by the coordinator's dispose; a rejection is
   * logged as a warning and never fails application teardown.
   * The coordinator captures dispose-time shutdown markers immediately before
   * this call for live capture; on-demand capture creates no ops records.
   * @returns resolves when the backend's pipeline has quiesced.
   */
  shutdown(): Promise<void>
}
```

`SessionTelemetryBackend` (`ctx.sessionTelemetry`, [firmas](#ctxsessiontelemetry--sessiontelemetrybackend-abstract-seam)) es la forma cargable del contrato —una implementación por contexto; cargar un duplicado lanza una excepción—, y un backend compone el `SessionTelemetryCoordinator` del seam en su constructor para instalar el lado de captura.

## El waterfall de expurgo: `session-telemetry/record`

Cada registro pasa por el [waterfall](../cordis-primer.es.md#cordis-waterfall-semantics) de `session-telemetry/record` entre la proyección y `emit()` ([entrada de evento](#session-telemetryrecord--waterfall)). El seam NO trae reglas propias: sin listener montado, los registros llegan al backend tal y como se capturaron, de modo que los datos exportados son exactamente tan limpios como las reglas que monte el despliegue. Los listeners se apilan transformando el valor de retorno de `next()`; devolver sin llamar a `next()` reemplaza todo lo que hay debajo; un listener que lanza una excepción retiene ese único registro en modo fail-closed dentro de la contención del coordinador. El expurgo se aplica solo a la copia exportada: el log canónico de la sesión nunca se reescribe.

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxsessiontelemetry--sessiontelemetrybackend-abstract-seam"></a>

### `ctx.sessionTelemetry` — `SessionTelemetryBackend` (abstract seam)

Loadable form of the backend contract: one implementation per context — the cordis `Service` registration under the `telemetry` key throws on a duplicate, cordis' standard behavior. A backend composes a SessionTelemetryCoordinator in its constructor to install the capture side.

```ts cordis-catalog
/**
 * See {@link SessionTelemetrySink.emit} — that declaration is the contract's one home.
 * @param record - the logical record to report; owned by the backend after the call.
 */
abstract emit(record: SessionTelemetryRecord): void

/** See {@link SessionTelemetrySink.flush}. */
flush?(): void

/**
 * See {@link SessionTelemetrySink.shutdown}.
 * @returns resolves when the backend's pipeline has quiesced.
 */
abstract shutdown(): Promise<void>
```

Source: [`packages/session/session-telemetry/src/index.ts`](../../packages/session/session-telemetry/src/index.ts)

<a id="session-telemetry-events"></a>

### `session-telemetry/*` events

<a id="session-telemetryrecord--waterfall"></a>

#### `session-telemetry/record` — waterfall

Transform one outbound record before it reaches the backend. This waterfall is the Service Definition's redaction extension point. It ships NO rules of its own: the innermost `next()` passes the record through unchanged, and with no listener mounted records reach the backend as captured, so exported data is exactly as clean as the rules a deployment mounts. Listeners stack by transforming `next()`'s return value; returning without `next()` replaces everything beneath. Dispatched synchronously on the capture hot path inside the coordinator's containment: a throwing listener withholds that one record (fail-closed) and never reaches the agent loop. Live capture dispatches at append time; on-demand capture dispatches while reading the canonical log. Redaction applies to the exported copy only; the canonical session log is never rewritten.

```ts cordis-catalog
/**
 * Transform one outbound record before it reaches the backend. This
 * waterfall is the Service Definition's redaction extension point. It ships NO rules
 * of its own: the
 * innermost `next()` passes the record through unchanged, and with no
 * listener mounted records reach the backend as captured, so exported
 * data is exactly as clean as the rules a deployment mounts. Listeners
 * stack by transforming `next()`'s return value; returning without
 * `next()` replaces everything beneath. Dispatched synchronously on the
 * capture hot path inside the coordinator's containment: a throwing
 * listener withholds that one record (fail-closed) and never reaches the
 * agent loop. Live capture dispatches at append time; on-demand capture
 * dispatches while reading the canonical log. Redaction applies to the
 * exported copy only; the canonical session log is never rewritten.
 * @param record - the candidate record, already the coordinator's own deep
 *   copy; listeners return a (possibly new) record and must not mutate it.
 * @mode waterfall
 */
'session-telemetry/record'(record: SessionTelemetryRecord, next: () => SessionTelemetryRecord): SessionTelemetryRecord
```

Source: [`packages/session/session-telemetry/src/index.ts`](../../packages/session/session-telemetry/src/index.ts)
<!-- END GENERATED cordis-surface -->
