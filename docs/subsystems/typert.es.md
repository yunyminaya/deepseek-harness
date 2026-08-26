# Llamadas remotas de Typert

[English](typert.md) | Español

Tipos compartidos por los artefactos Remote generados, el Host Gateway y los assemblies de API del Consumer. La [Agent Note del gateway Typert](../../.agents/notes/implemented/architecture/2026-08-02-typert-remote-method-calls.es.md) es dueña de las decisiones de arquitectura y de transporte; esta página registra los contratos públicos literales de [`dsh-typert-protocol`](../../packages/typert/protocol/src/types.ts) y [`dsh-api-gateway`](../../packages/api/gateway/src/types.ts).

## Declaraciones de Lookup y Context

Los paquetes de objetos de negocio extienden dos mapas vacíos mediante fusión de declaraciones. Un lookup asocia un tipo de objeto Host con su identidad wire; una declaración de Context asocia un tipo de Context con ámbito con su identidad wire. Los descriptores generados nombran estas claves, mientras que los providers del runtime aportan el comportamiento de resolución en vivo.

```ts type-equiv
/** Merge-extensible Host object lookup declarations. */
interface TypertLookupMap {}
```

```ts type-equiv
/** Merge-extensible scoped Context declarations. */
interface TypertContextMap {}
```

El registro conserva la declaración wire de un lookup después de que su resolver se descargue. Por eso la discovery SRC sigue clasificando el parámetro como lookup y falla con unavailable en lugar de aceptar el valor wire como un objeto de negocio ordinario.

```ts type-equiv
/** Stable wire declaration retained after a lookup provider unloads. */
interface TypertLookupDefinition {
  /** Merge-declared lookup key. */
  readonly key: string
  /** Source parameter name recognized by the SRC weak parser. */
  readonly parameter: string
  /** Wire field replacing the Host object parameter. */
  readonly wire: string
  /** Canonical Host type symbol used by strict generation. */
  readonly hostTypeSymbol: string
  /** Canonical wire type symbol used by strict generation. */
  readonly wireTypeSymbol: string
}
```

## Descriptores de invocación

Un `InvocationDescriptor` es reflexión local, no un mensaje wire. Las builds del Host y del Consumer generan descriptores correspondientes; la solicitud envía solo el endpoint y los `args` con nombre. Los codecs estrictos llevan schemas generados, mientras que los codecs SRC imponen valores seguros para JSON sin recuperación estructural de tipos. La cancelación es una señal de carrier fuera de banda que se inyecta después de los parámetros de negocio y nunca entra en `args`.

```ts type-equiv
/** Codec attached to one invocation parameter or result. */
type TypertCodec =
  | {
    readonly mode: 'strict'
    readonly typeSymbol: string
    readonly schema: TypertSchema
  }
  | {
    readonly mode: 'src-json'
  }
```

```ts type-equiv
/** One ordered business parameter in a Remote invocation. */
interface InvocationParameterDescriptor {
  /** Source-level parameter name. */
  readonly name: string
  /** Required key in the wire `args` object. */
  readonly wire: string
  /** Whether the value is JSON or requires a registered Host lookup. */
  readonly source: 'json' | 'lookup'
  /** Lookup key when `source` is `lookup`. */
  readonly lookup?: string
  /** Boundary codec for the wire representation. */
  readonly codec: TypertCodec
  /** Missing wire fields decode to `undefined` only for an explicitly declared `T | undefined`. */
  readonly acceptsUndefined?: true
}
```

```ts type-equiv
/** Carrier-independent description of one exported method invocation. */
interface InvocationDescriptor {
  /** Globally stable generated identity. */
  readonly id: string
  /** Cordis service key owning the method. */
  readonly service: string
  /** Wire namespace, defaulting to the service key. */
  readonly namespace: string
  /** Public instance method name. */
  readonly method: string
  /** Service member invoked when the exported method name is an alias. */
  readonly implementation?: string
  /** Receiver selection mode. */
  readonly invocation:
    | { readonly kind: 'direct' }
    | {
      readonly kind: 'context'
      readonly context: string
      readonly wire: string
      readonly codec: TypertCodec
    }
  /** Optional consuming-Context projection for one direct lookup parameter. */
  readonly scope?: {
    /** Context kind whose Client binder supplies the identity. */
    readonly context: string
    /** Lookup parameter wire field replaced by the Context identity. */
    readonly wire: string
  }
  /** Ordered business parameters. */
  readonly parameters: readonly InvocationParameterDescriptor[]
  /** Transport cancellation injected after business parameters instead of entering wire args. */
  readonly cancellation?: {
    /** Reserved final Host method parameter. */
    readonly parameter: 'signal'
  }
  /** Codec for the resolved method result. */
  readonly result: TypertCodec
  /** Source declaration used only for diagnostics. */
  readonly sourceLocation?: InvocationSourceLocation
}
```

## Registro de Typert

`ctx.typert` separa los descriptores del entorno actual, las contribuciones Remote seleccionadas explícitamente, los providers de lookup y los providers de Context con ámbito. Un provider de lookup es dueño de la declaración wire estable y del resolver por defecto; la composición del Host puede configurar para la misma clave un resolver síncrono o asíncrono con ámbito de effect, y descargar esa configuración restaura la política por defecto. Los registros son effects propiedad de Cordis y devuelven disposers que se pueden esperar.

```ts type-equiv
/** Minimal Typert runtime consumed through dependency inversion. */
interface TypertRegistryContract {
  readonly local: TypertLocalRegistry
  readonly remotes: TypertRemoteRegistry
  readonly lookups: TypertLookupRegistry
  readonly contexts: TypertContextRegistry
}
```

Las declaraciones de Consumer generadas fusionan los espacios de nombres directos en el mapa que hereda `TypertClientRemote`.

```ts type-equiv
/** Merge-extensible direct namespace surface generated for Client Remote services. */
interface TypertRemoteNamespaceMap {}
```

## Host Gateway

Connection decodifica el sobre de su carrier antes de llamar a `ctx.typertGateway`. La solicitud lleva campos wire exactos con nombre y, por separado, la señal de cancelación del carrier; los fallos de infraestructura y de frontera usan la taxonomía de errores en proceso del Gateway, las excepciones ordinarias las pliega el adaptador RPC en el código de error `internal` del transporte, y los errores RPC existentes que la política de lookup transporta a través de `TypertLookupFailure` se devuelven sin cambios.

```ts type-equiv
/** One Remote method request after a carrier has decoded its envelope. */
interface InvokeRemoteRequest {
  /** Remote namespace selected by the generated descriptor. */
  readonly namespace: string
  /** Exported Service method name. */
  readonly method: string
  /** Named wire values; fields must exactly match the descriptor. */
  readonly args: Readonly<Record<string, unknown>>
  /** Carrier or direct-caller cancellation injected only into cancellation-aware methods. */
  readonly signal?: AbortSignal
}
```

```ts type-equiv
/** Stable infrastructure and boundary failures emitted before or after business execution. */
type TypertGatewayErrorCode =
  | 'ambiguous-endpoint'
  | 'arguments-invalid'
  | 'binding-invalid'
  | 'context-failed'
  | 'context-not-found'
  | 'context-unavailable'
  | 'definition-unavailable'
  | 'input-invalid'
  | 'invocation-unavailable'
  | 'lookup-failed'
  | 'lookup-not-found'
  | 'lookup-unavailable'
  | 'method-unavailable'
  | 'provider-mismatch'
  | 'result-invalid'
  | 'service-unavailable'
  | 'signature-invalid'
```

```ts type-equiv
/** Host dispatcher consumed by Connection adapters. */
interface TypertGateway {
  /**
   * Invoke one live Remote method without assuming a carrier or response envelope.
   * @param request - decoded endpoint and named wire arguments.
   * @returns the validated business result.
   * @throws {@link TypertGatewayError} for dispatch, provider, or boundary failures; lookup-policy and business errors retain identity.
   */
  invoke(request: InvokeRemoteRequest): Promise<unknown>
}
```

## Consumer Remote

`ctx.remote` expone solo los espacios de nombres que aportan los artefactos `/remote` importados. `$mount()` instala los descriptores generados y los métodos concretos como una única operación propiedad del fiber. Cada espacio de nombres es un Service hijo de Cordis trazado `remote.<namespace>` cuyo ciclo de vida abarca sus métodos montados; ningún Proxy de JavaScript ni tipo de Service de negocio del Host entra en el Consumer.

```ts type-equiv
/** Client Remote capability implemented by the Gateway and consumed by Remote assemblies. */
interface TypertClientRemote extends TypertRemoteNamespaceMap {
  /**
   * Mount one generated Host-for-Client contribution in the caller's fiber.
   * @param contribution - explicitly selected Remote package artifact.
   * @returns disposer after namespace services and concrete methods are ready.
   */
  $mount(contribution: TypertRemoteContribution): Promise<TypertDisposer>
  /**
   * Subscribe to one forwarded Host event; delivery is one-way, in registration
   * order, and isolates a throwing listener from the rest.
   * @template Event - forwarded event name selected by the Host assembly.
   * @param event - forwarded Host event name, unchanged on the wire.
   * @param listener - receives the Host's argument list as declared by Cordis `Events`.
   * @returns disposer owned by the calling fiber.
   */
  $on<Event extends TypertRemoteEvent>(event: Event, listener: Events[Event]): () => void
  /**
   * Hand one decoded forwarded frame to the subscription table. The carrier
   * owning the Host frame sink calls this; a consumer subscribes with
   * {@link TypertClientRemote.$on} and never calls it.
   *
   * `event` is a plain string because this is the wire boundary: the name is
   * whatever the Host assembly's allowlist selected, and one nobody subscribed
   * to is dropped silently.
   * @param event - forwarded Host event name, exactly as the Host emitted it.
   * @param args - the Host argument list, already JSON-decoded.
   */
  $dispatch(event: string, args: readonly unknown[]): void
}
```

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxapiproxy--apiproxy"></a>

### `ctx.apiProxy` — `ApiProxy`

Root interface of the unified API. New client-request domain = one new file pair + one field here + one map row.

```ts cordis-catalog
/**
 * Response entry for server requests; not a domain method.
 * @param message - Client response carrying the server request's rpcId.
 * @returns Transport receipt for the response delivery.
 */
respond(message: ClientResponse): Promise<RpcReceipt>
```

Source: [`packages/host/apiproxy/src/api/index.ts`](../../packages/host/apiproxy/src/api/index.ts)

<a id="ctxtypert--typertregistry"></a>

### `ctx.typert` — `TypertRegistry`

Registry of generated schemas, package reflection, invocations, and Remote dependency providers.

```ts cordis-catalog
/**
 * Register one generated contribution atomically for the calling fiber.
 * Duplicate package-face identities, schemas, invocation ids, or endpoints
 * reject the whole batch.
 * @param contribution - generated schemas, reflection, and Host invocations.
 * @returns the exact effect disposer that removes this contribution.
 */
register(contribution: TypertContribution): TypertDisposer

/**
 * Look up one schema by `<package>#<name>`.
 * @param key - global schema key.
 * @returns the live schema record, or `undefined` when absent.
 */
get(key: string): TypertSchemaRecord | undefined

/**
 * Resolve one required schema.
 * @param key - global schema key.
 * @returns the live schema record.
 * @throws when the key is malformed, the package face is absent, or the schema is not contributed.
 */
resolve(key: string): TypertSchemaRecord

/**
 * Enumerate live schemas in registration order.
 * @param filter - optional package and face restriction.
 * @returns matching schema records.
 */
list(filter: TypertSchemaFilter = {}): TypertSchemaRecord[]

/**
 * Look up generated reflection for one package face.
 * @param packageName - exact npm package name.
 * @param face - face to query; defaults to the host runtime.
 * @returns the live package record, or `undefined` when absent.
 */
getPackage(packageName: string, face: TypertFace = 'host'): TypertPackageRecord | undefined

/**
 * Enumerate generated package reflection in registration order.
 * @param filter - optional package and face restriction.
 * @returns matching package records.
 */
listPackages(filter: TypertPackageFilter = {}): TypertPackageRecord[]

/**
 * Project a live Zod schema to JSON Schema without caching the result.
 * @param key - global schema key.
 * @param params - Zod projection parameters.
 * @returns a fresh JSON Schema document.
 */
toJSONSchema(key: string, params?: z.core.ToJSONSchemaParams): z.core.JSONSchema.BaseSchema
```

Types: [TypertContribution](invariants.es.md) · [TypertFace](invariants.es.md) · [TypertPackageFilter](invariants.es.md) · [TypertPackageRecord](invariants.es.md) · [TypertSchemaFilter](invariants.es.md) · [TypertSchemaRecord](invariants.es.md)

Source: [`packages/typert/registry/src/service.ts`](../../packages/typert/registry/src/service.ts)

<a id="ctxtypertgateway--typertgatewayservice"></a>

### `ctx.typertGateway` — `TypertGatewayService`

Resolve strict generated definitions or conservative SRC markers against current Cordis Services and Typert providers.

```ts cordis-catalog
/**
 * Invoke one live Remote method through strict generated reflection or SRC markers.
 * @param request - decoded endpoint and exact named wire arguments.
 * @returns the validated business result.
 * @throws {@link TypertGatewayError} for dispatch, provider, or boundary failures; lookup-policy and business errors retain identity.
 */
async invoke(request: InvokeRemoteRequest): Promise<unknown>
```

Source: [`packages/api/gateway/src/index.ts`](../../packages/api/gateway/src/index.ts)
<!-- END GENERATED cordis-surface -->
