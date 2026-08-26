# Acceso web

[English](web.md) | Español

El seam de acceso web —un [seam de capacidad](../../.agents/notes/implemented/architecture/2026-06-24-web-capability-seam.es.md) que abarca **dos operaciones** (search y fetch) en un único servicio `ctx.web`, repartido entre varios paquetes: Service Definition ([dsh-web](../../packages/web/web), `ctx.web` + los registros de providers), Service Providers ([dsh-web-search-exa](../../packages/web/web-search-exa), [dsh-web-search-perplexity](../../packages/web/web-search-perplexity), [dsh-web-search-deepseek](../../packages/web/web-search-deepseek), [dsh-web-fetch-http](../../packages/web/web-fetch-http)) y Consumer ([dsh-tool-web](../../packages/web/tool-web), los schemas de las herramientas `web_search`/`web_fetch`). Web es **una capacidad opcional**, no forma parte de la columna vertebral del agent loop (bucle del agente) —por eso su vocabulario vive aquí y no en [core.md](core.es.md). Cambiar de provider de búsqueda no cambia cómo pide el modelo una consulta, y cambiar de provider de fetch no cambia cómo pide el modelo una URL.

Fuente: [`packages/web/web/src/types.ts`](../../packages/web/web/src/types.ts)

## Por qué una capacidad tiene dos operaciones

Search y fetch no comparten schema de solicitud ni lógica de negocio, pero forman deliberadamente una única capa intermedia `ctx.web`: un solo dueño de la política de selección de provider, un solo vocabulario de abort/error y una única API de configuración orientada al producto del tipo «cómo llega este harness a la web». El coste son los pares de métodos paralelos `searchX`/`fetchX` del servicio; ese paralelismo es intencionado, no una extracción que faltara. Los providers registran **capacidades** (un `WebSearchProvider` o un `WebFetchProvider`), no herramientas; los nombres visibles para el modelo, los schemas, la guía de prompts y la presentación viven todos en el Consumer único `dsh-tool-web`.

## Solicitud y resultado de búsqueda

Cada solicitud seam lleva exactamente una `query`. El Consumer `dsh-tool-web` acepta un array `queries` obligatorio y lo reparte en solicitudes seam independientes; un array de un solo elemento realiza una búsqueda. `maxResults` es un límite propiedad del Consumer (la config `searchMaxResults` de `dsh-tool-web`, por defecto `8`) que atraviesa el seam y se aplica a la vuelta: si un provider devuelve de más, el seam trunca `sources[]` y establece `truncated`.

```ts type-equiv
/**
 * What one search-capable backend is asked to search. Each request carries one
 * query; a consumer may issue several requests. `maxResults` is a
 * `dsh-tool-web`-layer bound passed through unchanged and enforced on the way
 * back by the seam (see {@link WebSearchResult}).
 */
interface WebSearchRequest {
  readonly query: string
  /**
   * Upper bound on returned sources; the seam truncates to it. Omitted = no
   * bound. `dsh-tool-web` always sets it. A provider whose API supports a
   * result-count control (Exa's `numResults`) should apply it at the request
   * layer as a cost/latency optimization; the seam enforces the bound
   * regardless.
   */
  readonly maxResults?: number
}
```

```ts type-equiv
/**
 * Normalized search outcome. `content` is optional provider-generated answer
 * text or summary (Exa and DeepSeek return none; Perplexity returns a
 * generated answer).
 * `sources[]` is the portable citation shape. `truncated` is set by the seam
 * when it cut `sources[]` down to `maxResults`.
 */
interface WebSearchResult {
  /** Optional provider-generated answer text, search context, or summary. */
  readonly content?: string
  /** Citeable sources, already truncated to the request's `maxResults`. */
  readonly sources: readonly WebSearchSource[]
  /** True when the seam dropped sources to honor `maxResults`. */
  readonly truncated: boolean
}
```

```ts type-equiv
/**
 * One citeable source. A source always has a URL; `title`, `snippet`, and
 * `publishedAt` are optional because not every provider returns them — forcing
 * adapters to invent them would make the seam lie (Perplexity citations may be
 * URL-only). `dsh-tool-web` renders `title ?? hostname(url)` for display.
 */
interface WebSearchSource {
  readonly url: string
  readonly title?: string
  readonly snippet?: string
  /** Publication/crawl timestamp as a provider-supplied ISO-8601 string. */
  readonly publishedAt?: string
}
```

## Solicitud y resultado de fetch

```ts type-equiv
/**
 * What one fetch-capable backend is asked to retrieve. The request deliberately
 * omits timeout, format, prompt, and extraction controls: cancellation is a
 * direct execution argument, while presentation and higher-level LLM concerns
 * belong outside safe retrieval.
 */
interface WebFetchRequest {
  readonly url: string
}
```

El estado HTTP forma parte del estado del recurso obtenido, no es automáticamente un fallo: un fetch de red correcto de un `404`/`500` devuelve un `WebFetchResult` con el código de estado y un cuerpo decodificado limitado. `url` es la URL final tras las redirecciones permitidas. `WebError` está reservado para los fallos al recuperar o representar el recurso de forma segura.

```ts type-equiv
/**
 * Normalized fetch outcome. A successful network fetch of a non-2xx response is
 * a result, not an error: the status code is part of the fetched resource
 * state. {@link WebError} is reserved for failures to safely retrieve or
 * represent the resource.
 */
interface WebFetchResult {
  /** The final URL after allowed redirects (the request URL is in the request). */
  readonly url: string
  /** HTTP status code of the fetched response. */
  readonly statusCode: number
  /** Decoded body, classified by content kind. */
  readonly body: WebFetchBody
  /** True when the provider capped the decoded body. */
  readonly truncated: boolean
}
```

```ts type-equiv
/**
 * The decoded body of a fetched resource. A CLOSED discriminated union owned by
 * `dsh-web`: the provider decodes the kind and `dsh-tool-web` renders it, so a
 * new kind is a coordinated change across known packages, not a plugin
 * extension. Consumers `switch` on `kind` ending in `default: assertNever(...)`
 * so adding a kind breaks compilation at every consumer until handled. Each arm
 * stays its own object literal even where fields coincide, so an arm can gain
 * fields the others lack.
 */
type WebFetchBody =
  | { readonly kind: 'html'; readonly content: string }
  | { readonly kind: 'text'; readonly content: string }
```

## Disponibilidad de los providers

El `available(): boolean` de un provider es una comprobación LOCAL barata (presencia de credenciales, config parseable) y **no debe hacer llamadas de red**. Es una entrada para la selección en tiempo de ejecución, no un sistema de salud: `search()`/`fetch()` lo leen para elegir un provider utilizable, y un fallo de selección se manifiesta como el `WebError` estructurado sobre el que decide quien llama —que lleva el detalle sobre el que se puede ramificar (el id ausente o el conjunto de candidatos ambiguo) en su código y en su mensaje.

La selección nunca depende del orden de registro, config o HMR: una capacidad tiene un id de provider explícito (config `searchProvider`/`fetchProvider`, o la variable de entorno correspondiente que alimenta el mismo campo), o se autoselecciona cuando hay registrado exactamente un provider utilizable; varios providers utilizables sin id configurado es `WEB_PROVIDER_AMBIGUOUS`, no gana el primero.

## Errores

`WebError` extiende `HarnessError` (taxonomía de errores de [core.md](core.es.md)) con un `code: string` (abierto, como el error de cualquier otro seam —`LlmError`, `SubagentError`—), no una unión cerrada: un provider puede lanzar sus propios códigos sin tocar `dsh-web`, y los Consumers deben tolerar un código desconocido. Los códigos se reparten por dueño. Los códigos neutrales al seam los lanza el contrato compartido `WebRuntime`: `WEB_PROVIDER_UNAVAILABLE`, `WEB_PROVIDER_CONFIGURED_MISSING`, `WEB_PROVIDER_CONFIGURED_UNAVAILABLE`, `WEB_PROVIDER_AMBIGUOUS`, `WEB_DUPLICATE_PROVIDER` (un error de programación en tiempo de registro, análogo al `DUPLICATE_ADAPTER` de `LlmRuntime`), `WEB_ABORTED` y `WEB_PROVIDER_ERROR` (el comodín para el fallo propio de un provider que aflora a través del seam, incluido el fallo de red/transporte —DNS, conexión rechazada, TLS—). Los códigos de transporte de fetch son propiedad de la implementación `dsh-web-fetch-http` y un backend de fetch distinto no tiene por qué lanzarlos: `WEB_INVALID_URL`, `WEB_BLOCKED_URL`, `WEB_REDIRECT_BLOCKED`, `WEB_FETCH_TOO_LARGE`, `WEB_FETCH_TIMEOUT`, `WEB_UNSUPPORTED_CONTENT_TYPE`.

## El servicio

`WebRuntime` registra los providers de search y fetch, rechaza los ids duplicados con `WEB_DUPLICATE_PROVIDER` y resuelve los providers en tiempo de ejecución con errores de selección estructurados. El backend local de fetch solo acepta HTTP(S), rechaza las credenciales, limita las redirecciones, los bytes, los caracteres y el tiempo, revalida cada salto de redirección del mismo origen y decodifica el cuerpo; la presentación es cosa de la herramienta. El backend local no bloquea los destinos de redes privadas; no habilites `web_fetch` donde pueda alcanzar destinos internos sensibles.

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxweb--webruntime"></a>

### `ctx.web` — `WebRuntime`

The web access service. Registered as `ctx.web` (one instance per context).

Selection semantics (resolved at execution time, never order-dependent):

- A configured id that is registered and `available()` → that provider.
- A configured id not registered → `WEB_PROVIDER_CONFIGURED_MISSING`.
- A configured id registered but unavailable → `WEB_PROVIDER_CONFIGURED_UNAVAILABLE`.
- No id configured, exactly one registered usable provider → that provider.
- No id configured, multiple usable providers → `WEB_PROVIDER_AMBIGUOUS`.
- No id configured, no usable provider → `WEB_PROVIDER_UNAVAILABLE`.

```ts cordis-catalog
/**
 * Register a search provider. Throws {@link WebError} `WEB_DUPLICATE_PROVIDER`
 * if its id is already registered for search. Returns a disposer; disposed
 * with the calling fiber.
 * @param provider - the provider; its `id` is the registry key.
 * @returns the disposer that unregisters the provider.
 */
registerSearchProvider(provider: WebSearchProvider): () => void

/**
 * Register a fetch provider. Throws {@link WebError} `WEB_DUPLICATE_PROVIDER`
 * if its id is already registered for fetch. Returns a disposer; disposed
 * with the calling fiber.
 * @param provider - the provider; its `id` is the registry key.
 * @returns the disposer that unregisters the provider.
 */
registerFetchProvider(provider: WebFetchProvider): () => void

/**
 * Run one search through the selected provider. Resolves the provider at call
 * time with the selection rules above; throws {@link WebError} when the
 * capability cannot run. The seam enforces `request.maxResults` on the result:
 * if the provider over-returns, `sources[]` is truncated and `truncated` set.
 * @param request - the query and optional result limit.
 * @param signal - optional cancellation signal forwarded to the provider.
 * @returns the provider's results, capped to `request.maxResults`.
 */
async search(request: WebSearchRequest, signal?: AbortSignal): Promise<WebSearchResult>

/**
 * Retrieve one URL through the selected provider. Resolves the provider at
 * call time with the selection rules above; throws {@link WebError} when the
 * capability cannot run. A non-2xx response is a result, not a throw.
 * @param request - the URL plus retrieval options.
 * @param signal - optional cancellation signal forwarded to the provider.
 * @returns the retrieval outcome; non-2xx responses resolve descriptively.
 */
async fetch(request: WebFetchRequest, signal?: AbortSignal): Promise<WebFetchResult>
```

Source: [`packages/web/web/src/index.ts`](../../packages/web/web/src/index.ts)
<!-- END GENERATED cordis-surface -->
