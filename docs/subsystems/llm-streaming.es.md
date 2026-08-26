# Streaming de LLM (modelo de lenguaje de gran tamaño)

[English](llm-streaming.md) | Español

Los tipos de conversación y de streaming de [`packages/llm`](../../packages/llm/README.es.md): las variantes `Message`/`ContentBlock` que comparten todas las solicitudes y el historial duradero, la solicitud de modelo totalmente ensamblada, el protocolo en bruto `StreamChunk`, el contrato del adaptador que todo adaptador debe implementar y el ensamblador compartido. Los [paquetes core](core.es.md) conservan y registran estos valores en cada turno; esta página los declara.

Fuente: [`packages/llm/llm/src/types.ts`](../../packages/llm/llm/src/types.ts)

<a id="content-blocks-and-messages"></a>

## Bloques de contenido y mensajes

Una conversación se compone de `Message`s; un mensaje es una matriz de **bloques de contenido** tipados. La unión de bloques deriva de `ContentBlockMap`.

Fuente: [`packages/llm/llm/src/types.ts`](../../packages/llm/llm/src/types.ts)

```ts type-equiv
/**
 * Merge-extensible content blocks keyed by `type`. New core blocks must land
 * with adapter, UI, and compaction support.
 */
interface ContentBlockMap {
  'text': TextBlock
  'reasoning': ReasoningBlock
  'image': ImageBlock
  'tool-call': ToolCallBlock
  'tool-result': ToolResultBlock
}
```

Las interfaces de bloque (campos completos en el código fuente): `TextBlock` (`text`), `ReasoningBlock` (pensamiento, distinto del texto visible), `ImageBlock` (un [adjunto de imagen](attachment.es.md) duradero), `ToolCallBlock` (`id: CallId`, `name`, `arguments` JSON en bruto) y `ToolResultBlock` (`toolCallId`, `content: ContentBlock[]` anidado, `isError?`). `ContentBlock = ContentBlockMap[ContentBlockType]`. Una nueva modalidad pertenece al mapa extensible por fusión solo cuando su adaptador, su interfaz, su compactación y sus vías de replay duradero la respetan.

Fuente: [`packages/llm/llm/src/message.ts`](../../packages/llm/llm/src/message.ts)

Un `Message` es un valor identificado e inmutable de rol/fuente/contenido. Los mensajes de asistente producidos por el modelo nombran el provider y el modelo que los produjeron y portan datos de replay opcionales privados del adaptador en su source:

```ts type-equiv
/** Provider/model identity and adapter-private replay data for an assistant message. */
interface AssistantProvenance {
  /** Provider route that produced the message. */
  provider: string
  /** Provider model id that produced the message. */
  model: string
  /**
   * Lossless-JSON adapter state needed to replay the provider response.
   * `LlmRuntime` exposes it to a target adapter only when that adapter instance
   * currently owns both this historical provider and the target provider.
   */
  replayState?: unknown
}
```

```ts type-equiv
/** One immutable message representation shared by delivery, durable history, and model requests. */
interface Message {
  /** Stable identity preserved across every representation boundary. */
  readonly id: MessageId
  /** Provider-neutral conversation role. */
  readonly role: 'system' | 'user' | 'assistant'
  /** Exact model-facing blocks. */
  readonly content: ContentBlock[]
  /** Required source fields supplied by the producer. */
  readonly source: MessageSource
}
```

De dónde proviene un mensaje es en sí mismo un tipo suma extensible por fusión:

```ts type-equiv
/**
 * Where a message (or injected content) came from.
 * Merge-extensible sum type — plugins add their own `kind`s.
 */
interface MessageSourceMap {
  user: { kind: 'user' }
  plugin: { kind: 'plugin'; plugin: string } & ContextFormed
  model: ModelMessageSource
  tool: ToolMessageSource
}
```

La identidad del productor y la forma de presentación son independientes. `kind` responde *quién produjo esto*; el `form` opcional responde *qué tipo de información es*, y los Consumers deciden cómo presentarla. Varios productores pueden compartir una misma forma, y un productor puede emitir más de una forma a lo largo de una sesión. Los valores son semánticos y crecen de uno en uno; un valor ausente o no reconocido usa el valor por defecto documentado y se presenta como contenido opaco:

```ts type-equiv
/**
 * The kind of information in producer-supplied context, declared by the
 * producer beside its provenance.
 *
 * `MessageSource.kind` answers *who produced this*; `form` answers *what kind
 * of thing it is*, and the two axes are deliberately independent — several
 * producers share one form, and one producer may emit more than one form over
 * a session.
 *
 * The vocabulary is SEMANTIC, never visual: a value states that the content is
 * a file's instructions or a catalog of available items, and a consumer decides
 * what that looks like. Colors, icons, ordering, and collapse defaults are the
 * consumer's business and must not enter this union. It grows one value at a
 * time as producers gain the structured fields their form needs; an absent or
 * unknown value is the documented default, presented as opaque content.
 */
type ContextForm =
  /** Instructions read out of workspace files the model is expected to follow. */
  | 'instructions'
  /** A catalog of items available in this session, republished as it changes. */
  | 'catalog'
  /** Current state, where a later snapshot from the same producer supersedes an earlier one. */
  | 'snapshot'
  /** A one-off account of something that just happened; it supersedes nothing. */
  | 'notice'
  /** A message another agent addressed to this one. */
  | 'relay'
  /** Material lifted out of another session's log, possibly reduced on the way in. */
  | 'recall'
```

```ts type-equiv
/** One named contribution to a `snapshot`-form context, in assembly order. */
interface ContextSnapshotSection {
  /** The contributing subsystem's name. */
  readonly name: string
  /** That contribution's model-facing text, exactly as assembled. */
  readonly text: string
}
```

```ts type-equiv
/**
 * Producer-declared {@link ContextForm} and the fields that form requires,
 * mixed into the source types that carry one.
 *
 * Discriminated by `form` so a producer cannot select a form without the
 * fields needed to present it: a `notice` must record its one-line
 * account, a `snapshot` its sections. Omitting `form` stays valid — an
 * undeclared context is the documented default.
 */
type ContextFormed =
  | { readonly form?: never }
  | { readonly form: 'instructions' }
  | { readonly form: 'catalog' }
  | {
    readonly form: 'snapshot'
    /** The named contributions this snapshot assembled, in order. */
    readonly sections: readonly ContextSnapshotSection[]
  }
  | {
    readonly form: 'notice'
    /** One-line account of what happened, shown without expanding the row. */
    readonly summary: string
  }
  | { readonly form: 'relay' }
  | { readonly form: 'recall' }
```

<a id="streamchunk--the-raw-protocol"></a>

## `StreamChunk` — el protocolo en bruto

Una respuesta en streaming entremezcla varios bloques tipados (texto, razonamiento, varias llamadas de herramienta). `index` vincula cada delta a su bloque; `block-end` porta el `ContentBlock` totalmente ensamblado para que los Consumers no tengan que reensamblar los deltas por su cuenta. Es una unión discriminada **cerrada**: un `switch` sobre `type` termina con `assertNever`, de modo que añadir una variante rompe la compilación en todos los Consumers que deban manejarla.

```ts type-equiv
/**
 * Adapter-private lossless-JSON state for replaying a successful response,
 * carried by a terminal `finish` chunk and stored on the assembled assistant
 * message's model source. Both halves stay opaque to the harness; only the
 * split is shared vocabulary, so assembly can keep stored metadata aligned
 * with stored content without reading either half.
 */
interface ReplayEnvelope {
  /** Response-level adapter-private metadata (ids, native stop reason). */
  response: unknown
  /**
   * Per-block adapter-private metadata, one entry per emitted block in
   * first-seen stream order. When assembly drops a block it drops the entry at
   * the same position; entries whose length does not match the emitted block
   * count discard the whole envelope. An adapter whose metadata is independent
   * of block structure omits this field and the envelope passes through
   * assembly unchanged.
   */
  blocks?: readonly unknown[]
}
```

```ts type-equiv
/**
 * Raw streaming protocol emitted by adapters.
 * Block indexes correlate interleaved deltas, and `block-end` carries the
 * assembled block. Adapters emit usage before the terminal finish and nothing
 * afterward; tool arguments remain raw JSON strings. An adapter implementation
 * may throw, but `LlmRuntime.stream()` normalizes that failure to a terminal
 * `error` or `aborted` finish before exposing it to consumers.
 */
type StreamChunk =
  | { type: 'block-start'; index: number; blockType: ContentBlockType }
  | { type: 'text-delta'; index: number; text: string }
  | { type: 'reasoning-delta'; index: number; text: string }
  | { type: 'tool-call-delta'; index: number; id: CallId; name?: string; argumentsDelta: string }
  | { type: 'block-end'; index: number; block: ContentBlock }
  | { type: 'usage'; usage: TokenUsage }
  | {
    type: 'finish'
    reason: FinishReason
    /** Replay metadata for a successful response; see {@link ReplayEnvelope}. */
    replayState?: ReplayEnvelope
  }
```

## `LlmFailure`

Todo fallo final del adaptador, lanzado o en banda, se normaliza en un único payload serializable y neutral respecto al provider. `providerRetryAfterMs` es un retraso positivo validado solicitado por el provider, no una decisión de reintento; `ProviderRequestId` es una cadena opaca con marca con fines de diagnóstico.

```ts type-equiv
/** Serializable provider or transport failure facts; policy decides whether they are retryable. */
interface LlmFailure {
  /** Human-readable provider or transport failure. */
  readonly message: string
  /** Stable provider-neutral machine-routing code. */
  readonly code: string
  /** HTTP status returned by the provider, when available. */
  readonly status?: number
  /** Provider-requested delay in milliseconds, when valid and available. */
  readonly providerRetryAfterMs?: number
  /** Opaque provider-issued request identifier for diagnostics. */
  readonly requestId?: ProviderRequestId
}
```

## El contrato del adaptador

Todo adaptador DEBE cumplirlas, y todo Consumer puede confiar en ellas:

- **`usage` antes de `finish`, nada después de `finish`.** Diferir ambos hasta el marcador de fin de stream del provider para que un chunk final solo de uso no pueda violar el orden.
- **Los `arguments` de las llamadas de herramienta siguen siendo cadenas JSON en bruto de principio a fin.** Los fragmentos parciales se transmiten vía `argumentsDelta`; un provider que devuelve objetos ya analizados los vuelve a serializar en `block-end`.
- **Dos vías de error autorizadas, un único tipo `LlmFailure`.** Un fallo puede LANZARSE desde `stream()` (errores de transporte/protocolo) **o** terminar el stream con `finish {kind:'error'|'aborted', failure}` (errores en banda del provider, para adaptadores que no pueden lanzar a mitad del stream). `LlmError.failure` porta el mismo `LlmFailure`. Tras seleccionar el adaptador de la llamada, el stream conserva el objeto `Error` lanzado exacto y asocia a esa llamada los hechos inmutables y la política de reintentos inmutable del registro que la atiende; el agent loop (bucle del agente) cierra el paso fallido y ofrece el error, los hechos, los hechos reintentados previamente inmutables, la política de servicio y la señal de turno a `agent/request-error`. Un oyente de manejo devuelve `{ kind: 'retry' }` tras completar su reparación; sin recuperación, el fallo estructurado se convierte en el error de turno y no se confirma ningún mensaje de asistente normal ni efecto secundario de herramienta para ese intento.
- **Una llamada de adaptador es un intento de provider.** Los adaptadores desactivan los reintentos de la librería. La recuperación a nivel de agent abre otro turno numerado duradero; los llamadores directos de `ctx.llm.stream()` siguen siendo de un solo intento.
- **Los estancamientos del provider están acotados en el transporte.** Ambos adaptadores remotos distribuidos exponen un `streamIdleTimeoutMs` positivo finito con un valor por defecto de cinco minutos. El perro guardián solo se arma mientras la llamada `next()` del iterador esté pendiente, usa una única señal estable para toda la solicitud, asigna su propia caducidad a `TIMEOUT` y conserva una cancelación previa del llamador como `ABORTED`.
- **El desbordamiento de contexto tiene un único código canónico.** Ambos adaptadores de DeepSeek clasifican el detalle explícito del provider mediante `isContextWindowExceededError()` y exponen `CONTEXT_WINDOW_EXCEEDED`, tanto si el fallo llega como un `LlmError` HTTP lanzado como si llega como error de finish en banda. Los Consumers enrutan según el código, nunca según el texto del provider.
- **Una finalización vacía es un error reintentable, no un éxito silencioso.** Ambos adaptadores asignan un finish `stop` final sin bloques de contenido a `finish {kind:'error'}` con el código canónico `EMPTY_RESPONSE`, y `dsh-llm-retry` lo reintenta por defecto; ver [las respuestas de modelo vacías son reintentables](../../.agents/notes/implemented/bug-fix/2026-07-24-empty-model-response-is-retryable.es.md).
- **Toda solicitud HTTP al provider porta la cabecera de atribución de la aplicación.** Los adaptadores envían `attributionHeaders()` (más abajo) — la línea base de `User-Agent` — y lo demuestran con una prueba a nivel de cable.
- **El estado de replay es propiedad del adaptador; su división es compartida.** Un `finish` con éxito puede portar un `ReplayEnvelope`: metadatos opacos a nivel de respuesta más entradas por bloque opcionales alineadas con la secuencia de bloques emitida. La alineación es el vocabulario del harness: cuando el ensamblado descarta un bloque, descarta la entrada en la misma posición, de modo que los metadatos almacenados siempre describen el contenido almacenado. El bucle almacena el envelope podado junto con el mensaje de asistente ensamblado. En una solicitud posterior, `LlmRuntime` solo pasa el estado cuando el provider histórico y el provider de destino están registrados en ese momento en exactamente la misma instancia de adaptador. Ese adaptador valida el estado y es dueño de cualquier conversión entre modelos o entre providers; los demás adaptadores reciben el contenido neutral respecto al provider y los campos de provider/modelo sin el estado privado. El contenido duradero sigue siendo la fuente de autoridad: un estado almacenado que el adaptador lector no puede usar degrada ese único mensaje a una conversión neutral respecto al provider con un diagnóstico, en lugar de fallar la solicitud.

## `ResolvedRetryPolicy`

La configuración de reintentos se resuelve antes del registro de la ruta en una unión discriminada inmutable. El modo normal porta `mode: 'normal'`, un `maxRetries` finito, `retryableCodes` y los obligatorios `initialDelayMs`, `maxDelayMs` y `jitterRatio`; el modo siempre porta `mode: 'always'` y los mismos campos de backoff obligatorios sin máximo finito. Omitir una política de provider usa el valor por defecto normal de cinco reintentos. Los ajustes en capas pueden conservar un `maxRetries` o `retryableCodes` solo de modo normal tras cambiar al modo siempre; el resolver ignora esos campos inactivos y captura la política pura de siempre. `LlmRuntime.providerRetryPolicy(provider)` devuelve el valor registrado, y `llmRetryPolicyOf(stream)` devuelve el valor capturado del registro que atiende la llamada una vez que esta lo selecciona, de modo que una liberación o un reemplazo posterior de la ruta no pueda cambiar la política de recuperación de un fallo en curso. El [catálogo de configuración generado](../config-catalog.es.md) enumera los campos de entrada opcionales.

## `AppIdentity` — atribución de la aplicación

La identidad pública estática de la aplicación que todo adaptador envía a los providers ([`packages/llm/llm/src/attribution.ts`](../../packages/llm/llm/src/attribution.ts)). `attributionHeaders(identity?)` la asigna únicamente a la cabecera estándar `User-Agent`; las cabeceras de atribución de aplicación específicas de OpenRouter no están soportadas intencionadamente por este contrato. El `APP_IDENTITY` por defecto obtiene su versión del manifest (manifiesto) del paquete; cada campo es un hecho público del producto — sin secretos, rutas, ids de sesión ni identificadores por usuario, y nada por solicitud puede influir en los valores. Fundamento: [Atribución obligatoria de `User-Agent`](../../.agents/notes/implemented/architecture/2026-06-21-mandatory-app-attribution-headers.es.md).

```ts type-equiv
/**
 * Static public application identity sent to LLM providers.
 *
 * Every field is a public product fact, safe on every request: no secrets,
 * local paths, session ids, prompt text, or per-user identifiers belong here,
 * and nothing per-request may influence the values.
 */
interface AppIdentity {
  /** `User-Agent` product token (lowercase, hyphenated). */
  product: string
  /** Product version; sourced from package metadata, never hand-copied. */
  version: string
  /** Repository home URL of the app, used as the `User-Agent` comment. */
  url: string
}
```

## `TokenUsage`

Contabilización de tokens por llamada. Los conteos son **disjuntos**: `inputTokens` es solo la entrada sin caché; la entrada en caché se informa por separado, y la entrada facturada es la suma de los tres. Los adaptadores cuyos providers integran los aciertos de caché en un único total de prompt (el `prompt_tokens` de DeepSeek) los restan de nuevo. `reasoningTokens`, cuando está presente, es un detalle informativo ya incluido en `outputTokens`; los totales no deben sumarlo otra vez.

```ts type-equiv
/**
 * Token accounting for one model call (cache fields are optional).
 *
 * Counts are DISJOINT: `inputTokens` is uncached input only; cached input is
 * reported separately as `cacheReadTokens`/`cacheWriteTokens` (billed input =
 * sum of the three). Adapters whose providers fold cache hits into a total
 * prompt count (DeepSeek's `prompt_tokens`) subtract them out.
 */
interface TokenUsage {
  inputTokens: number
  outputTokens: number
  cacheReadTokens?: number
  cacheWriteTokens?: number
  reasoningTokens?: number
}
```

## `BlockAssembler`

`BlockAssembler` ([`packages/llm/llm/src/assembler.ts`](../../packages/llm/llm/src/assembler.ts)) es la única implementación compartida que recompone un stream de `StreamChunk` en `ContentBlock`s, uso, razón de finish y estado de replay. El bucle registra los chunks en bruto mientras alimenta los mismos chunks a través de un ensamblador, y después almacena el contenido de asistente ensamblado con el provider y el modelo que lo produjeron. Un Consumer que necesite el resultado ensamblado sin reimplementar la recomposición usa esto.

Una única decisión de conservar/descartar cubre contenido y metadatos a la vez: un finish de `max-tokens` descarta toda llamada de herramienta porque una llamada truncada no es segura de ejecutar, y la misma decisión poda la entrada por bloque del envelope de replay en cada posición descartada. Por tanto, `blocks()` y `replayState` no pueden discrepar, sea cual sea lo que el ensamblado elimine.

```ts public-api
/**
 * Incrementally assembles raw {@link StreamChunk}s into complete
 * {@link ContentBlock}s and a final assistant {@link Message}.
 *
 * The agent loop feeds it while logging raw chunks for replay fidelity, then
 * reads `blocks()` / `message()` / `usage` / `finish` once the stream ends,
 * or `interruptedBlocks()` when cancellation cut the stream short.
 *
 * Tolerant of delta-only protocols (no block-start/end); deltas arriving for
 * an index already closed by `block-end` are ignored (malformed stream) so a
 * misbehaving adapter cannot grow memory or corrupt a completed block.
 */
declare class BlockAssembler {
  /**
   * Feed one chunk into the assembly state.
   * @param chunk - the next raw chunk, in stream order.
   */
  push(chunk: StreamChunk): void;
  /**
   * Assemble all blocks seen so far, in stream order.
   * @returns one block per seen index, except that max-token truncation drops
   *   tool calls that cannot be executed safely; an open block assembles from
   *   its accumulated deltas (an unknown block type never closed by `block-end` throws).
   */
  blocks(): ContentBlock[];
  /**
   * Assemble the prefix an interrupted stream can safely finalize: closed and
   * open text/reasoning blocks with non-whitespace content, in stream order.
   * Tool calls are omitted because interruption precedes dispatch; retaining
   * one would require a fabricated result. Open unknown blocks are also omitted.
   * @returns the kept blocks; empty when nothing streamed before the interruption.
   */
  interruptedBlocks(): ContentBlock[];
  /** Usage from the `usage` chunk; undefined until one arrives. */
  get usage(): TokenUsage | undefined;
  /** Finish reason from the `finish` chunk; `{kind: 'stop'}` when the stream ended without one. */
  get finish(): FinishReason;
  /**
   * Replay metadata from the terminal finish chunk, if any, with per-block
   * entries pruned in step with {@link blocks}. Undefined when the envelope's
   * entries do not align with the emitted blocks.
   */
  get replayState(): ReplayEnvelope | undefined;
  /**
   * The assembled assistant message.
   * @param source - producer attribution for the assembled message.
   * @returns a frozen assistant-role message over `blocks()` (same open-block assembly rules).
   */
  message(source: MessageSource = { kind: 'plugin', plugin: 'dsh-llm/assembler' }): Message;
}
```

<a id="the-model-request-and-result"></a>

## La solicitud de modelo

Una llamada de modelo es un `GenerateOptions` totalmente ensamblado. El adaptador responde con un stream en bruto de [`StreamChunk`](#streamchunk--the-raw-protocol); el Consumer lo ensambla con [`BlockAssembler`](#blockassembler).

Fuente: [`packages/llm/llm/src/types.ts`](../../packages/llm/llm/src/types.ts)

El descubrimiento de providers y modelos usa pequeños descriptores neutrales respecto al provider. Un catálogo de modelos es orientativo: el enrutamiento sigue teniendo como clave un provider registrado, y un adaptador puede aceptar ids de modelo no listados.

Registrar un adaptador devuelve un identificador: el disposer (función de liberación), más el reemplazo atómico de rutas que necesita un plugin cuyo conjunto de rutas es configurable por el usuario.

```ts type-equiv
/**
 * What {@link LlmRuntime.registerAdapter} returns: the disposer, plus an
 * atomic route replacement for the same adapter instance.
 */
interface AdapterRegistrationHandle {
  /** Release every route this registration currently holds. */
  (): void
  /**
   * Replace this registration's routes with `providers`, keeping the same
   * adapter instance. The candidate set is validated in full first — a
   * conflict with another adapter, an invalid name, or bad provider metadata
   * throws and leaves the current routes untouched — and the swap itself is
   * one synchronous section, so no request can observe a gap. An empty array
   * is legal here (a settings section that emptied holds zero routes while
   * staying registered), unlike an empty initial registration.
   *
   * Throws `LlmError` with code `REGISTRATION_DISPOSED` once the registration
   * has been released: its routes are gone and its disposer has already run,
   * so anything registered afterwards would have no owner left to release it.
   * @param providers - the complete next route set for this registration.
   */
  replace(providers: string[]): void
}
```

```ts type-equiv
/** Display metadata for one registered provider route. */
interface LlmProviderInfo {
  /** Provider route key used by {@link GenerateOptions.provider}. */
  id: string
  /** Human-readable provider name for selectors and diagnostics. */
  name: string
}
```

Los plugins de adaptador declaran además qué rutas *podrían* funcionar mediante `registerConfigurableProviders()`, dirigiendo la sección de ajustes de usuario de cada una, para que las superficies de configuración puedan ofrecer providers latentes antes de que se registre cualquier ruta.

```ts type-equiv
/**
 * One provider route an adapter plugin can activate through configuration,
 * whether or not the route is currently registered. Configuration surfaces
 * merge this directory with `listProviders()` to offer every configurable
 * provider alongside its live/dormant state.
 */
interface LlmConfigurableProvider {
  /** Provider route key this entry activates when configured. */
  provider: string
  /** Human-readable provider name for configuration surfaces. */
  displayName: string
  /** User-settings namespace whose section configures this provider. */
  settingsNs: string
  /**
   * Path from that namespace's section root to this provider's profile
   * object; empty when the whole section is the profile.
   */
  settingsPath: readonly string[]
  /**
   * Whether the owning adapter knows this route only because configuration
   * declared it — a gateway or self-hosted server it ships nothing about.
   * Absent means the adapter draws no such distinction; false means it does
   * and this route is one of its own. Only the adapter can answer: a stored
   * profile is how a user-added route AND a corrected shipped one both look
   * from outside.
   */
  declared?: boolean
}
```

```ts type-equiv
/** One adapter-discovered model; catalog membership is advisory, not request validation. */
interface LlmModelInfo {
  /** Provider route that owns this model entry. */
  provider: string
  /** Model id passed to {@link GenerateOptions.model}. */
  id: string
  /** Human-readable model name for selectors. */
  name: string
  /** Optional user-facing distinction from otherwise similar models. */
  description?: string
  /** Accepted request modalities; absent means unknown, while an explicit omission is negative capability. */
  inputModalities?: readonly ModelModality[]
}
```

Los metadatos sensibles a la corrección se resuelven aparte del catálogo orientativo y son propiedad del adaptador que atiende la ruta exacta. La capacidad de contexto, los valores por defecto de la llamada del adaptador y las elecciones de razonamiento comparten un único resultado de modelo exacto para que los Consumers no repitan la resolución autoritativa del modelo.

```ts type-equiv
/** Provider-owned context capacity for one exact provider/model route. */
interface LlmModelContext {
  /** Maximum combined request and response context in tokens. */
  contextWindow: number
}
```

El esfuerzo de razonamiento es otra capacidad de ruta exacta. El core marca los identificadores pero no enumera sus valores; cada adaptador es dueño del conjunto ordenado, de los nombres mostrados y del valor por defecto de despliegue opcional.

```ts type-equiv
/** Adapter-owned identifier for one model's selectable reasoning effort. */
type ReasoningEffortId = Branded<'ReasoningEffortId'>
```

```ts type-equiv
/** Display metadata for one adapter-owned reasoning effort. */
interface LlmReasoningEffortInfo {
  /** Opaque stable value accepted by {@link GenerateOptions.reasoningEffort}. */
  id: ReasoningEffortId
  /** Human-readable effort name for selectors and diagnostics. */
  name: string
  /** Optional user-facing distinction from otherwise similar efforts. */
  description?: string
}
```

```ts type-equiv
/** Selectable reasoning efforts for one exact provider/model route. */
interface LlmModelReasoningInfo {
  /** Supported efforts in adapter-preferred display order. */
  efforts: readonly LlmReasoningEffortInfo[]
  /**
   * Adapter-configured default materialized into requests when callers omit
   * an effort. Absence preserves the provider's own default.
   */
  defaultEffort?: ReasoningEffortId
}
```

```ts type-equiv
/** Exact-route model metadata resolved by its owning adapter. */
interface LlmResolvedModelInfo extends LlmModelInfo {
  /** Provider-owned context capacity when known. */
  context?: LlmModelContext
  /** Adapter-configured per-request output cap materialized when callers omit one. */
  defaultMaxTokens?: number
  /** Adapter-owned selectable reasoning levels when exposed. */
  reasoning?: LlmModelReasoningInfo
}
```

```ts type-equiv
/** A single model request, fully assembled. */
interface GenerateOptions {
  /** Registered provider route selecting the adapter instance. */
  provider: string
  model: string
  /** Adapter-owned reasoning effort selected for this exact model. */
  reasoningEffort?: ReasoningEffortId
  /**
   * Ordered conversation messages, exactly as the provider sees them (after
   * the `system` slot). A loop-built request assembles them as
   * the derived history (dsh-agent-loop); a hand-built one-shot passes any list.
   */
  messages: Message[]
  /** System prompt text (adapters map to the provider's system slot). */
  system?: string
  /** Tool schemas (adapters map to the provider's `tools` field). */
  tools?: ToolSchema[]
  temperature?: number
  maxTokens?: number
  /**
   * Stop sequences: generation halts as soon as the model produces any one of
   * these strings (adapters map to the provider's stop field, e.g. OpenAI
   * `stop`). The stop string itself is not included in the output.
   */
  stop?: string[]
  signal?: AbortSignal
  /**
   * Session identity stamped by the loop for request routing. Replay uses it
   * to separate cursors; adapters may map it to model-hidden transport metadata.
   */
  sessionId?: Branded<'SessionId'>
  /**
   * Provider-neutral classification for an auxiliary model call. Adapters may
   * map the purpose to model-hidden transport metadata or purpose-specific
   * generation policy. Ordinary conversation requests leave it unset.
   */
  purpose?: 'compaction' | 'session-title'
}
```

Por qué se detuvo una respuesta de modelo es una razón extensible por fusión. Los fallos finales del provider portan el [`LlmFailure`](#llmfailure) del contrato de streaming:

```ts type-equiv
/**
 * Why a model response stopped.
 * Merge-extensible so adapters can surface provider-specific reasons.
 */
interface FinishReasonMap {
  'stop': { kind: 'stop' }
  'tool-calls': { kind: 'tool-calls' }
  'max-tokens': { kind: 'max-tokens' }
  'aborted': { kind: 'aborted'; failure: LlmFailure }
  'error': { kind: 'error'; failure: LlmFailure }
}
```

`FinishReason = FinishReasonMap[keyof FinishReasonMap]`. `TokenUsage` (contabilización por llamada con campos de caché disjuntos) se detalla [más abajo](#tokenusage).

`GenerateOptions.tools` porta `ToolSchema` — la descripción JSON Schema de una herramienta, tal como se envía al modelo. Se declara en dsh-llm (no en dsh-tools) precisamente porque forma parte de la solicitud que el bucle ensambla en cada paso:

```ts type-equiv
/**
 * JSON-schema description of a tool, as sent to the model.
 *
 * Declared here (not in dsh-tools) because it is part of {@link GenerateOptions};
 * dsh-tools' ToolDefinition and dsh-system-prompt's PromptAssembly both import
 * it from this package.
 */
interface ToolSchema {
  name: string
  description: string
  /** JSON Schema object for the arguments. */
  parameters: Record<string, unknown>
}
```

El `ToolSchema` orientado al modelo es el tipo de wire; el `ToolDefinition` registrado que lo produce (schema + `execute`) está en [tools.md](tools.es.md).

Un provider que una superficie aún está redactando no tiene ruta ni catálogo, por lo que la interrogación se describe por separado: la solicitud porta el borrador que el usuario está editando, y la respuesta son candidatos que una superficie puede adoptar, no un catálogo que deba servir.

```ts type-equiv
/**
 * One interrogation of a provider endpoint that configuration has not stored
 * yet. Configuration surfaces send the draft a user is still editing, so the
 * request carries the endpoint and credential directly instead of naming a
 * route: a provider being added has no route to name.
 */
interface LlmModelDiscoveryRequest {
  /**
   * Route the draft is editing, when it edits an existing one. A route whose
   * adapter already knows its models answers from that knowledge instead of
   * asking the endpoint — the adapter's own registry is the better answer, and
   * it costs no network call.
   */
  provider?: string
  /**
   * Endpoint to interrogate. Optional because a route the adapter already
   * describes needs none; a route it does not must supply one.
   */
  baseURL?: string
  /** Wire protocol the endpoint speaks, when the draft names one. */
  api?: string
  /** Credential for this interrogation alone; the harness never stores it. */
  apiKey?: string
  /** Caller cancellation; implementations must settle promptly after it aborts. */
  signal?: AbortSignal
}
```

```ts type-equiv
/**
 * One model an endpoint reports about itself. Every field but the id is
 * optional because most provider listings disclose an id and nothing else;
 * a surface adopting one of these still owes the capacities its adapter needs.
 */
interface LlmDiscoveredModel {
  /** Model id the endpoint accepts. */
  id: string
  /** Human-readable name when the endpoint supplies one. */
  name?: string
  /** Maximum combined request and response context, when disclosed. */
  contextWindow?: number
  /** Maximum output tokens, when disclosed. */
  maxTokens?: number
}
```

### La envoltura de la solicitud: `LlmCallConfig` y la cabecera registrada

El bucle construye cada solicitud a partir del estado registrado. `EpochHeader` registra la configuración de la llamada, marca los campos aportados por los valores por defecto del adaptador y registra el prompt renderizado y el orden autoritativo de herramientas devuelto (configurado por `toolOrder`, o lexicográfico si no está definido) mediante instantáneas completas de `request/header`. Junto con el historial derivado, esto hace que la solicitud sea reconstruible a partir del registro de sesión. Ver [session.md](session.es.md#the-request-header-event-requestheader) y la [Agent Note sobre reconstruibilidad](../../.agents/notes/implemented/architecture/2026-07-05-reconstructable-requests.es.md).

`agent/request` recibe una semilla de configuración de llamada congelada y puede devolver un reemplazo para cambiar el provider, el modelo, el esfuerzo de razonamiento o el muestreo. Antes del waterfall (cascada de eventos), el bucle elimina los valores marcados como valores por defecto del adaptador para que la preparación de modelo exacto materialice los valores actuales de la ruta seleccionada; los ajustes explícitos sin marcar permanecen en la propuesta. Tras el waterfall, la preparación rechaza los ids de esfuerzo explícitos no soportados sin recortarlos y registra la configuración efectiva más los campos aportados por los valores por defecto del adaptador bajo la señal de turno. La llamada preparada conserva un único registro de adaptador durante todo el despacho. Las solicitudes que llegan a `llm/stream` están profundamente congeladas, de modo que la mutación lanza una excepción, y portan una identidad de bucle local al proceso para que los observadores no confundan las llamadas auxiliares congeladas registradas por separado con las solicitudes de conversación.

En el cable, una solicitud construida por el bucle lee el slot del `system` (el prompt renderizado y ensamblado) seguido del historial derivado. La instantánea de solicitud registrada termina con el `user/message` más reciente en el primer paso de un turno y con los resultados de las llamadas de herramienta del paso anterior en los pasos siguientes. El invariante de dev recalcula exactamente esta ecuación contra cada solicitud construida por el bucle.

FIXME(call-config-shape): revisar qué campos restantes son realmente de nivel de época a efectos de caché (`model` y el esfuerzo de razonamiento propiedad del modelo son explícitos; los escalares de muestreo están aquí por precaución).

```ts type-equiv
/**
 * Provider, model, reasoning effort, and sampling scalars of one conversation's
 * requests. Every field maps 1:1 onto the same-named `GenerateOptions` field;
 * the loop builds requests from the logged header rather than accepting these
 * per call.
 */
interface LlmCallConfig {
  provider: string
  model: string
  reasoningEffort?: ReasoningEffortId
  temperature?: number
  maxTokens?: number
  stop?: string[]
}
```

```ts type-equiv
/**
 * Effective config fields supplied by exact-model adapter resolution rather
 * than by the caller's request proposal.
 */
interface LlmCallConfigAdapterDefaults {
  reasoningEffort?: true
  maxTokens?: true
}
```

## Contratos de servicio y provider

`LlmAdapter` es el contrato del provider: crea una subclase, implementa `stream()` y registra una instancia de adaptador con `ctx.llm.registerAdapter(providers, adapter)`. `GenerateOptions.provider` selecciona el adaptador registrado; `GenerateOptions.model` se pasa a ese adaptador y no necesita estar registrado al inicio del ciclo de vida. Las rutas de provider duplicadas fallan de forma atómica. El `providerRetryPolicy()` opcional se captura por ruta con los valores por defecto normales, mientras que `providerInfo()` y el `listModels()` asíncrono alimentan `LlmRuntime.listProviders()` / `listModels()` con metadatos de selector desacoplados. Ese catálogo es orientativo, no una lista blanca de solicitudes: el adaptador sigue siendo la fuente de autoridad y puede aceptar ids de modelo no listados. Una consulta asíncrona `resolveModel()` devuelve la identidad exacta del modelo más una capacidad de contexto opcional sensible a la corrección, un `defaultMaxTokens` configurado por el adaptador e ids de razonamiento ordenados propiedad del modelo con un valor por defecto de despliegue opcional; los campos ausentes significan metadatos no disponibles o comportamiento propiedad del provider, no una pertenencia inválida al catálogo. El resolver recibe una cancelación opcional y debe resolverse con prontitud tras la cancelación. `LlmRuntime.resolveModelInfo()` valida y desacopla el agregado. En la frontera final del adaptador, `resolveCallConfig()` materializa el valor por defecto de salida solo cuando `maxTokens` está ausente, y valida y materializa el razonamiento, de modo que las llamadas directas no pueden eludir ninguno de los comportamientos configurados; el despacho directo captura un registro antes de esperar esa resolución. El agent loop (bucle del agente) usa en su lugar `prepareCall()` para conservar el mismo registro a través de la resolución del modelo, el registro de la cabecera duradera y el despacho, retener los metadatos de contexto desacoplados de esa búsqueda exacta e informar de qué campos de configuración estableció el adaptador por defecto. La búsqueda del adaptador ocurre en la continuación final del waterfall `llm/stream`, de modo que un oyente puede cortocircuitar la llamada o enrutar una solicitud desechable mutable antes de la búsqueda. AgentLoop observa un intento de solicitud una vez que el waterfall exterior devuelve un identificador de stream; esa frontera limitada no demuestra que se haya construido un adaptador final perezoso o que haya iniciado la I/O del provider. La correlación de `index` de `block-start` / `block-end` junto con el ensamblador significan que un adaptador solo tiene que emitir chunks bien formados — el reensamblado de bloques no es problema de cada adaptador. [architecture.md](../architecture.es.md#turn-flow) muestra dónde se sitúan `ctx.llm.stream()` y el waterfall `llm/stream` en un turno.

```ts type-equiv
/** One model call whose config and adapter registration were resolved together. */
interface PreparedLlmCall {
  /** Detached, deep-frozen config with any adapter-owned default materialized. */
  readonly config: LlmCallConfig
  /** Immutable retry policy captured with the adapter registration. */
  readonly retryPolicy: ResolvedRetryPolicy
  /** Detached context metadata resolved with the registration-bound call. */
  readonly context?: LlmModelContext
  /** Exact model modalities captured with the adapter dispatch generation. */
  readonly inputModalities?: readonly ModelModality[]
  /** Config fields materialized by the captured adapter rather than proposed by the caller. */
  readonly adapterDefaults: LlmCallConfigAdapterDefaults
  /**
   * Dispatch this call once through the registration captured during
   * preparation. The request's call-config fields must match {@link config};
   * reuse or mismatch fails with `INVALID_PREPARED_CALL`.
   * @param options - fully assembled request carrying the prepared config.
   * @returns the chunk stream, including the `llm/stream` waterfall.
   */
  stream(options: GenerateOptions): AsyncIterable<StreamChunk>
}
```

```ts public-api
/**
 * Provider-wire adapter for the harness message and stream vocabulary. Register implementations
 * with `ctx.llm.registerAdapter(providers, adapter)`. Every provider HTTP request must include
 * `attributionHeaders()`; prove the headers are added in the wire request or library header hook. The direct-fetch
 * DeepSeek and library-backed pi-ai adapters meet this contract through different internals.
 */
declare abstract class LlmAdapter {
  /**
   * Describe one provider route owned by this adapter.
   * @param provider - a route passed to `registerAdapter()` for this instance.
   * @returns detached display metadata whose id must equal `provider`.
   */
  providerInfo(provider: string): LlmProviderInfo;
  /**
   * Return the provider-owned retry policy captured with this route.
   * @param _provider - a route passed to `registerAdapter()` for this instance.
   * @returns a resolved policy, or `undefined` to use the normal defaults.
   */
  providerRetryPolicy(_provider: string): ResolvedRetryPolicy | undefined;
  /**
   * List models this adapter can currently advertise for one owned provider.
   * The result is advisory: an adapter may accept unlisted model ids, and
   * consumers must not turn absence into request rejection.
   * @param _provider - one provider route owned by this adapter.
   * @returns discoverable models in adapter-preferred order.
   */
  listModels(_provider: string): Promise<readonly LlmModelInfo[]>;
  /**
   * Resolve all metadata available for one exact model. This query is
   * independent of the advisory catalog and does not validate request routing.
   * @param provider - one provider route owned by this adapter.
   * @param model - exact model id passed to {@link GenerateOptions.model}.
   * @param _signal - cancellation for this exact-model lookup; asynchronous
   *   implementations must settle promptly after it aborts.
   * @returns provider/model identity plus any context, call-default, and reasoning metadata.
   */
  resolveModel(
    provider: string,
    model: string,
    _signal?: AbortSignal,
  ): Promise<LlmResolvedModelInfo>;
  /**
   * Bind exact model metadata and the eventual request dispatch to one adapter generation.
   * Dynamic adapters override this so settings changes between preparation and
   * dispatch cannot combine one generation's capabilities with another's endpoint.
   * @param provider - registered provider route.
   * @param model - exact model id.
   * @param signal - cancellation for model resolution.
   * @returns model metadata and a one-generation stream entry point.
   */
  async prepareCall(provider: string, model: string, signal?: AbortSignal): Promise<PreparedAdapterCall>;
  /**
   * Stream one model call as raw chunks. The only required method.
   * @param options - the fully-assembled request; implementations must honor `options.signal`.
   * @returns the chunk stream, obeying the adapter contract documented on `StreamChunk`.
   */
  abstract stream(options: GenerateOptions): AsyncIterable<StreamChunk>;
}
```

`ContentBlockType` (el conjunto de claves que portan los bloques correlacionados por `index`) deriva del [`ContentBlockMap`](#content-blocks-and-messages) de más arriba.

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generado a partir del código fuente por `scripts/gen-cordis-catalog.ts` (cuya actualidad verifica `pnpm run verify-cordis-catalog` en doc-sync (sincronización de documentación); regenerar con `pnpm run gen-cordis-catalog`) — los lados lingüísticos solo difieren en las rutas de documentos emparejados específicas de cada idioma. Los bloques de firma usan un bloque `ts cordis-catalog` y conservan el JSDoc original del código fuente; los modos de despacho se definen en el [primer](../cordis-primer.es.md#dispatch-modes), y la API `ctx` heredada del framework vive en [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxllm--llmruntime"></a>

### `ctx.llm` — `LlmRuntime`

El servicio `llm` abstracto: un registro de adaptadores más una API de llamadas de modelo en streaming, interceptable mediante el waterfall `llm/stream`.

```ts cordis-catalog
/**
 * Register an adapter for the given provider routes. Throws `LlmError` with code
 * `DUPLICATE_ADAPTER` if any provider already has an adapter (all-or-nothing).
 * Disposed with the fiber.
 * @param providers - every provider route this adapter should serve.
 * @param adapter - the adapter that streams calls for those providers.
 * @returns the disposer, carrying {@link AdapterRegistrationHandle.replace}.
 */
registerAdapter(providers: string[], adapter: LlmAdapter): AdapterRegistrationHandle

/**
 * Describe provider routes with a registered adapter.
 * @returns detached provider metadata in registration order.
 */
listProviders(): LlmProviderInfo[]

/**
 * Declare provider routes an adapter plugin can activate through
 * configuration. Registration is all-or-nothing: an empty list, invalid
 * entry, or a provider already declared by any registration throws
 * `LlmError` without registering the rest. Disposed with the fiber.
 * @param entries - every configurable provider this plugin owns.
 * @returns a handle that withdraws all of them, and can atomically replace them.
 */
registerConfigurableProviders(entries: readonly LlmConfigurableProvider[]): DirectoryRegistrationHandle

/**
 * List every declared configurable provider, registered or dormant.
 * @returns detached directory entries in declaration order.
 */
listConfigurableProviders(): LlmConfigurableProvider[]

/**
 * Offer to interrogate provider endpoints on behalf of the settings
 * namespace this plugin owns. The namespace is the key because that is what
 * a configuration surface already holds from the configurable-provider
 * directory, and because a provider being *added* has no route to name yet.
 * Disposed with the fiber.
 * @param settingsNs - the namespace whose profiles this discovery serves.
 * @param discover - interrogates one endpoint; must honor `request.signal`.
 * @returns the disposer that withdraws the offer.
 */
registerModelDiscovery( settingsNs: string, discover: (request: LlmModelDiscoveryRequest) => Promise<readonly LlmDiscoveredModel[]>, ): () => void

/**
 * Interrogate one provider endpoint for the models it advertises. The
 * request describes a draft, not a stored route, so nothing here reads or
 * writes settings or credentials — the caller owns both, and the reply is
 * candidate metadata a surface may offer for adoption.
 * @param settingsNs - namespace whose registered discovery serves this draft.
 * @param request - the endpoint, protocol, and one-shot credential to use.
 * @returns the advertised models, deduplicated in endpoint order.
 */
async discoverModels( settingsNs: string, request: LlmModelDiscoveryRequest, ): Promise<LlmDiscoveredModel[]>

/**
 * Resolve the retry policy captured when one provider route was registered.
 * @param provider - registered provider route to inspect.
 * @returns the provider-owned policy, with normal defaults already resolved.
 */
providerRetryPolicy(provider: string): ResolvedRetryPolicy

/**
 * Discover models advertised by one registered provider. Catalog membership
 * is advisory and never changes routing or request validation.
 * @param provider - registered provider route to inspect.
 * @returns detached model metadata in adapter-preferred order.
 */
async listModels(provider: string): Promise<LlmModelInfo[]>

/**
 * Resolve and validate all metadata from the adapter that owns one exact
 * route. The result is detached from adapter-owned objects; catalog
 * membership remains advisory and does not control request routing.
 * @param provider - registered provider route to inspect.
 * @param model - exact model id passed to the adapter.
 * @param signal - optional cancellation for adapter-owned asynchronous lookup.
 * @returns exact model identity plus available context and reasoning metadata.
 */
async resolveModelInfo( provider: string, model: string, signal?: AbortSignal, ): Promise<LlmResolvedModelInfo>

/**
 * Validate a conversation call config against its exact model capability and
 * materialize adapter-configured defaults. Unsupported explicit efforts
 * reject before provider I/O; no clamping or aliasing is performed. This
 * standalone query does not bind a later dispatch; use {@link prepareCall}
 * when logging and streaming must share one adapter registration.
 * @param config - provider/model route and optional request controls.
 * @param signal - optional cancellation for adapter-owned capability lookup.
 * @returns a detached config only when a default must be materialized.
 */
async resolveCallConfig(config: LlmCallConfig, signal?: AbortSignal): Promise<LlmCallConfig>

/**
 * Resolve one call under its current adapter registration. The returned
 * one-shot handle keeps that registration across header logging and dispatch,
 * so HMR cannot combine one adapter's capability result with another adapter.
 * @param config - provider/model route and optional request controls.
 * @param signal - optional cancellation for adapter-owned capability lookup.
 * @returns a prepared config and its registration-bound stream entry point.
 */
async prepareCall(config: LlmCallConfig, signal?: AbortSignal): Promise<PreparedLlmCall>

/**
 * Stream one model call as raw chunks (token-level deltas). Replay state is
 * retained only when the same adapter instance owns its historical provider
 * and the target provider. Final adapter selection remains fixed through
 * asynchronous exact-model resolution and dispatch. Adapter selection,
 * dispatch, and iteration failures become terminal `error` or `aborted`
 * finish chunks; middleware, nested-call, cleanup, and consumer failures
 * remain thrown.
 * @param options - the full request; `options.provider` selects the adapter.
 * @returns the chunk stream, possibly wrapped by `llm/stream` listeners.
 */
stream(options: GenerateOptions): AsyncIterable<StreamChunk>
```

Fuente: [`packages/llm/llm/src/index.ts`](../../packages/llm/llm/src/index.ts)

<a id="llm-events"></a>

### Eventos `llm/*`

<a id="llmadapters-updated--emit"></a>

#### `llm/adapters-updated` — emit

La topología de providers cambió: un adaptador registró o anuló el registro de rutas, o el directorio de providers configurables ganó o perdió entradas. Esta notificación de registro sin payload se dispara en cada punto de confirmación (incluida la liberación de registros); los Consumers releen `listProviders()`, `listModels()` o `listConfigurableProviders()` para conocer el nuevo estado. Los fallos de los observadores están contenidos y no pueden vetar la mutación del registro.

```ts cordis-catalog
/**
 * The provider topology changed: an adapter registered or unregistered
 * routes, or the configurable-provider directory gained or lost entries.
 * This payload-free registry notification fires at each commit point
 * (including registration disposal); consumers re-read `listProviders()`,
 * `listModels()`, or `listConfigurableProviders()` for the new state.
 * Observer failures are contained and cannot veto the registry mutation.
 * @mode emit
 */
'llm/adapters-updated'(): void
```

Fuente: [`packages/llm/llm/src/types.ts`](../../packages/llm/llm/src/types.ts)

<a id="llmstream--waterfall"></a>

#### `llm/stream` — waterfall

Waterfall en torno a cada llamada de modelo en streaming (reintento, replay, enrutamiento). Vinculado al `LlmRuntime`; llama a `next()` para llegar al stream del adaptador resuelto, o produce tus propios chunks para cortocircuitarlo.

```ts cordis-catalog
/**
 * Waterfall around every streaming model call (retry, replay, routing).
 * Bound to the {@link LlmRuntime}; call `next()` to reach the resolved
 * adapter's stream, or yield your own chunks to short-circuit.
 * @param options - the full request. A LOOP-built request carries the
 *   process-local {@link markAgentLoopRequest} identity and arrives deep-frozen
 *   (mutation throws): its content is a pure function of the session log (the
 *   reconstructability Agent Note), so listeners read it, never rewrite it.
 *   Hand-built calls do not carry that marker; their messages already obey
 *   the immutable creation contract.
 * @mode waterfall
 */
'llm/stream'(this: LlmRuntime, options: GenerateOptions, next: () => AsyncIterable<StreamChunk>): AsyncIterable<StreamChunk>
```

Fuente: [`packages/llm/llm/src/index.ts`](../../packages/llm/llm/src/index.ts)
<!-- END GENERATED cordis-surface -->
