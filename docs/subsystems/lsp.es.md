# Navegación LSP

[English](lsp.md) | Español

El seam LSP — un [seam de capacidad](../../.agents/notes/implemented/architecture/2026-07-15-lsp-capability-seam.es.md) que expone la navegación semántica de código en un único servicio `ctx.lsp`, repartido entre varios paquetes: la Service Definition ([dsh-lsp](../../packages/lsp/lsp), `ctx.lsp` + el registro de providers), un Service Provider genérico ([dsh-lsp-stdio](../../packages/lsp/lsp-stdio), un host de language server stdio configurado) y el Consumer ([dsh-tool-lsp](../../packages/lsp/tool-lsp), el schema de la herramienta `lsp`). LSP es **una capacidad opcional**, no forma parte de la columna vertebral del agent loop — así que su vocabulario vive aquí, no en [core.md](core.es.md). Un cambio de provider no cambia cómo el modelo pide la navegación.

Fuente: [`packages/lsp/lsp/src/types.ts`](../../packages/lsp/lsp/src/types.ts)

## Operaciones y coordenadas

El seam y el modelo exponen exactamente cuatro consultas semánticas; la unión es cerrada, así que añadir una es un cambio impuesto por el compilador en todo el seam, los providers y la herramienta. Las posiciones y los rangos son UTF-16 basados en cero, en línea con el protocolo; la herramienta orientada al modelo es dueña de la convención de cursor basada en uno y convierte a la entrada y a la salida.

```ts type-equiv
/**
 * The four semantic queries the seam and model expose. A closed union: adding an operation is a
 * compile-enforced change across the seam, providers, and the tool. Symbols and call hierarchy are
 * not operations here; they need different schemas.
 */
type LspOperation = 'goToDefinition' | 'findReferences' | 'goToImplementation' | 'hover'
```

```ts type-equiv
/** A zero-based UTF-16 cursor coordinate, matching the LSP wire convention. */
interface LspPosition {
  /** Zero-based line. */
  readonly line: number
  /** Zero-based UTF-16 code-unit offset within the line. */
  readonly character: number
}
```

```ts type-equiv
/** A zero-based UTF-16 half-open range `[start, end)`. */
interface LspRange {
  readonly start: LspPosition
  readonly end: LspPosition
}
```

## Solicitud

Todos los campos son obligatorios: `workspaceRoot` lo suministra el llamante, `languageId` proviene del registro del provider (no de la solicitud), y los consumidores son dueños de los timeouts y los límites de resultados — así que ningún campo necesita valores por defecto de implementación y no hay paso `resolve()`. El provider recibe la solicitud del llamante más el `languageId` derivado, que solo sincroniza el documento transitorio y nunca participa en la selección.

```ts type-equiv
/**
 * A caller's normalized query. Every field is required: `workspaceRoot` is caller-supplied,
 * `languageId` comes from the provider registration (not here), and consumers own timeouts and
 * result limits — so no field needs implementation defaulting and there is no `resolve()` step.
 */
interface LspQueryRequest {
  /** Which semantic query to run. */
  readonly operation: LspOperation
  /** The source file to query (relative to `workspaceRoot` or absolute; the provider canonicalizes). */
  readonly filePath: string
  /** The zero-based UTF-16 cursor position to query at. */
  readonly position: LspPosition
  /** The workspace root the provider resolves against and indexes; required, never defaulted. */
  readonly workspaceRoot: string
}
```

```ts type-equiv
/**
 * A request as a provider receives it: the caller's {@link LspQueryRequest} plus the `languageId`
 * the seam derived from the provider's extension mapping. The language id only synchronizes the
 * transient document; it does not participate in selection.
 */
interface LspProviderQuery extends LspQueryRequest {
  /** The LSP language id for `filePath`, from this provider's extension mapping. */
  readonly languageId: string
}
```

## Resultado

Una unión discriminada CERRADA: las operaciones de navegación se normalizan en `locations`, `hover` en contenido o `null`. Los consumidores hacen `switch` sobre `kind` hasta el agotamiento, de modo que un brazo nuevo rompe la compilación hasta que se maneja. `findReferences` siempre incluye las declaraciones — el provider lo impone internamente, así que los llamantes no reciben ninguna bandera. La variante `locations` lleva `resolvedWorkspaceUri`, el URI `file:` canónico del workspace del provider. Un llamante que relativiza los URIs de ubicación usa esa coordenada en lugar de aplicar las reglas de ruta de la plataforma anfitriona a la raíz de la solicitud, posiblemente con symlinks.

```ts type-equiv
/** One resolved location: a document URI and the range within it. */
interface LspLocation {
  /** The target document URI (`file:` or otherwise), verbatim from the server. */
  readonly uri: string
  /** The range within the target document. */
  readonly range: LspRange
}
```

```ts type-equiv
/** Normalized hover content, or `null` for no hover at the position. */
interface LspHover {
  /** The normalized hover text (markdown or plaintext, provider-joined). */
  readonly contents: string
  /** The range the hover applies to, when the server supplied one. */
  readonly range?: LspRange
}
```

```ts type-equiv
/**
 * The closed result union. Navigation operations (`goToDefinition`, `findReferences`,
 * `goToImplementation`) normalize to `locations`; `hover` normalizes to content or `null`.
 * Consumers `switch` on `kind` to exhaustiveness so a new arm breaks compilation until handled.
 *
 * The `locations` variant carries `resolvedWorkspaceUri`: the provider's canonical `file:` URI for
 * the request's workspace root. A caller that relativizes location URIs MUST use this, not parse the
 * request's possibly symlinked process path with host-platform rules; the execution platform may
 * differ from the caller's.
 */
type LspQueryResult =
  | { readonly kind: 'locations'; readonly locations: readonly LspLocation[]; readonly resolvedWorkspaceUri: string }
  | { readonly kind: 'hover'; readonly hover: LspHover | null }
```

## Provider y servicio

Un provider es dueño de un `id` con marca estable y un mapa de extensiones exclusivo en minúsculas con punto inicial. `registerProvider` reserva el id y cada extensión de forma atómica — un registro inválido o conflictivo no publica nada — y su disposer libera todas las reservas. La selección es por consulta e independiente del orden; sin coincidencia lanza `LspError` `LSP_UNAVAILABLE`. El seam no expone tipos de protocolo, controles de proceso/documento ni una vía de escape JSON-RPC genérica.

```ts type-equiv
/**
 * A language-server backend registered on `ctx.lsp`. Each provider owns a stable {@link
 * LspProviderId} and an extension-to-language-id map (lowercase, leading-dot keys).
 * `findReferences` always includes declarations — the provider enforces this internally; callers
 * get no flag.
 */
interface LspProvider {
  /** Stable provider identity, reserved atomically with the extension mappings. */
  readonly id: LspProviderId
  /** Lowercase leading-dot extension → LSP language id (e.g. `{ '.ts': 'typescript' }`). */
  readonly extensionToLanguage: Readonly<Record<string, string>>
  /**
   * Run one query. The seam has already selected this provider and derived `languageId`.
   * @param request - the resolved provider query (caller request + derived language id).
   * @param signal - optional cancellation; the provider stops its own work when it aborts.
   * @returns the normalized, closed-union result.
   */
  query(request: LspProviderQuery, signal?: AbortSignal): Promise<LspQueryResult>
}
```

```ts type-equiv
/**
 * The LSP capability seam (`ctx.lsp`). Owns provider registration/selection and normalized query
 * execution; exposes exactly the four operations and no protocol escape hatch.
 */
interface LspService {
  /**
   * Register a provider, atomically reserving its id and every normalized extension. Any conflict
   * or invalid input publishes nothing and throws `LspError`; the returned disposer releases all
   * reservations. Disposed with the calling fiber.
   * @param provider - the backend to register.
   * @returns a synchronous disposer releasing the id and all extension reservations.
   */
  registerProvider(provider: LspProvider): () => void
  /**
   * Select a provider by the file's extension and run one query. Selection is per-query and
   * order-independent; no match throws `LspError` `LSP_UNAVAILABLE`.
   * @param request - the normalized query.
   * @param signal - optional cancellation forwarded to the selected provider.
   * @returns the normalized, closed-union result.
   */
  query(request: LspQueryRequest, signal?: AbortSignal): Promise<LspQueryResult>
}
```

`LspProviderId` es el id con marca del seam (`Branded<'LspProviderId'>` de [dsh-brand](../../packages/util/brand)); `LspError` extiende `HarnessError` con códigos estables como `LSP_INVALID_PROVIDER`, `LSP_CONFLICT`, `LSP_UNAVAILABLE`, `LSP_DISPOSED`, `LSP_UNSUPPORTED_OPERATION` y `LSP_MALFORMED_RESPONSE`, que los llamantes usan para enrutar en lugar de parsear `message`.

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.es.md).

<a id="ctxlsp--lspservice"></a>

### `ctx.lsp` — `LspService`

The LSP capability seam (`ctx.lsp`). Owns provider registration/selection and normalized query execution; exposes exactly the four operations and no protocol escape hatch.

```ts cordis-catalog
/**
 * Register a provider, atomically reserving its id and every normalized extension. Any conflict
 * or invalid input publishes nothing and throws `LspError`; the returned disposer releases all
 * reservations. Disposed with the calling fiber.
 * @param provider - the backend to register.
 * @returns a synchronous disposer releasing the id and all extension reservations.
 */
registerProvider(provider: LspProvider): () => void

/**
 * Select a provider by the file's extension and run one query. Selection is per-query and
 * order-independent; no match throws `LspError` `LSP_UNAVAILABLE`.
 * @param request - the normalized query.
 * @param signal - optional cancellation forwarded to the selected provider.
 * @returns the normalized, closed-union result.
 */
query(request: LspQueryRequest, signal?: AbortSignal): Promise<LspQueryResult>
```

Source: [`packages/lsp/lsp/src/types.ts`](../../packages/lsp/lsp/src/types.ts)
<!-- END GENERATED cordis-surface -->
