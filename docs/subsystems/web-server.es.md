# Servidor HTTP

[English](web-server.md) | Español

[dsh-host-webserver](../../packages/host/webserver) es el carrier HTTP de navegador para el host de la GUI: un único plugin de `node:http` que proporciona `ctx.webServer`, un registro de rutas con nombre, callbacks de transformación de index.html y un handler de fallback que un plugin puede reclamar. No forma parte del agent loop (bucle del agente) ni es un seam de capacidad; no conoce ningún concepto del harness, y otro plugin registra todas las rutas de funcionalidad, incluido el puente `/api`, los bundles de plugin y el flujo de eventos HMR (hot module replacement) ([nota sobre capas](../../.agents/notes/implemented/architecture/2026-07-19-gui-layering-and-rpc-protocol.es.md)). Solo sirve a navegadores: Electron carga los archivos compilados mediante `file://` y envía las peticiones fetch a través de un puente IPC en lugar de este servidor.

Fuente: [`packages/host/webserver/src/index.ts`](../../packages/host/webserver/src/index.ts)

## Rutas

```ts type-equiv
/** Route match kind: 'exact' matches the pathname verbatim; 'prefix' p matches p and p/<anything>. */
type WebRouteKind = 'exact' | 'prefix'
```

```ts type-equiv
/** One named route registration. */
interface WebRoute {
  kind: WebRouteKind
  /** Absolute pathname, no trailing slash. */
  path: string
  /** Owns the full response lifecycle (may hold the response open, e.g. SSE). */
  handler: (req: IncomingMessage, res: ServerResponse) => void | Promise<void>
}
```

El orden de coincidencia es fijo: primero la tabla de coincidencias exactas, después el prefijo coincidente más largo y por último el fallback registrado. El orden de registro no tiene semántica visible para las peticiones: las rutas con nombre se componen para ser disjuntas, y el asiento de fallback responde a cualquier cosa que ninguna ruta con nombre reclame; solo hay un dueño, y un segundo registro lanza una excepción. La composición Web distribuida ocupa el asiento con [`dsh-host-frontend-static`](../../packages/host/frontend-static/src/index.ts), el servidor SPA de dist con semántica fija: lo que no sea GET/HEAD responde 405, un traversal fuera de la raíz de dist responde 403, un index legible se renderiza en la raíz de dist y en la ruta de index configurada, los archivos existentes se sirven directamente, los destinos ausentes o que no son archivos responden 404 vacío y las extensiones desconocidas se envían como octet-stream.

## Configuración

```ts type-equiv
/** Gateway config: the listen address. */
interface Config {
  /** Listen host; the two supported values are loopback and all-interfaces. */
  host: '127.0.0.1' | '0.0.0.0'
  /** Listen port; zero requests an OS-assigned port. */
  port: number
}
```

`host` solo acepta `127.0.0.1` (postura predeterminada) y `0.0.0.0` (exposición deliberada a la red); no hay TLS, autenticación ni política de origen, así que un bind fuera de loopback expone el servidor a esa red. La ubicación de dist es un hecho de ensamblado del plugin de frontend que ocupa el asiento.

## El servicio

`WebServer` (`ctx.webServer`) empieza a escuchar inmediatamente al activarse; un fallo de escucha (EADDRINUSE…) rechaza la inicialización y el proceso de arranque informa del fiber que falló. `register(route)` añade una ruta con nombre y devuelve su disposer; un `(kind, path)` duplicado lanza una excepción porque los patrones de ruta son un contrato a nivel de composición y una colisión es una mala configuración. `collectIndexInjections()` reúne las filas estructuradas `IndexInjection` a través de un emit de `webserver/index-inject`, y `renderIndex(html)` las renderiza en las respuestas de index raíz y configurado correctas antes de aplicar en orden de registro las transformaciones de vía de escape `tapIndex(transform)` en crudo; [dsh-client-modules](../../packages/client/modules) responde al evento con las filas del manifest de arranque. `port` lee el puerto en escucha, incluido el puerto asignado por el sistema operativo cuando `config.port` es 0.

Una petición cuyo manejo lanza una excepción (un escape % malformado que golpea `decodeURIComponent`, un cliente que se desconecta a mitad del cuerpo) se registra como advertencia y se responde 400 —o se destruye el socket si las cabeceras ya se enviaron—; nunca se cierra el proceso. El disposal empareja `close()` con `closeAllConnections()` porque un handler puede mantener abierta su respuesta (SSE, Server-Sent Events) y esas conexiones nunca terminan por sí solas; sin el cierre forzado, el teardown se colgaría. El paquete nunca imprime: la línea de la URL es cosa de la shell. El detalle operativo por paquete, incluido el pipeline de observación de bundles en modo desarrollo, queda en el [README](../../packages/host/webserver/README.md).

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxwebserver--webserver"></a>

### `ctx.webServer` — `WebServer`

The browser HTTP carrier service. Activation listens immediately. Route registration order does not affect requests because configured named routes must be distinct, and the fallback handler answers anything not yet claimed during startup with 404 until its owner registers. A listen failure rejects initialization, and the boot process reports the failed fiber.

```ts cordis-catalog
/**
 * Register a named route. Duplicate (kind, path) throws — route patterns are
 * a composition-level contract, so a collision is a misconfiguration.
 * @param route - kind, path, and the owning handler.
 * @returns the disposer removing the route.
 */
register(route: WebRoute): () => void

/**
 * Register an exact-path HTTP upgrade route. Duplicate paths throw because
 * one socket can have only one protocol owner.
 * @param route - pathname and handler owning negotiation plus socket use.
 * @returns the disposer removing the route.
 */
registerUpgrade(route: WebUpgradeRoute): () => void

/**
 * Claim the fallback seat: the handler answering every request no named
 * route matches (the SPA dist server in the shipped Web composition). One
 * owner only — a second registration throws, because two fallbacks cannot
 * compose.
 * @param handler - owns the full response lifecycle of unmatched requests.
 * @returns the disposer releasing the seat.
 */
registerFallback(handler: WebRoute['handler']): () => void

/**
 * Register a raw-HTML index transform, the escape hatch for markup no
 * {@link IndexInjection} row expresses: {@link renderIndex} applies taps in
 * registration order after rendering the structured rows.
 * @param transform - pure html-to-html function.
 * @returns the disposer removing the transform.
 */
tapIndex(transform: (html: string) => string): () => void

/**
 * Run an index.html body through the registered taps in registration order
 * — called by the fallback owner on every index response it renders.
 * @param html - the raw index.html body.
 * @returns the transformed body.
 */
applyIndexTaps(html: string): string

/**
 * Gather the structured injection table: one `webserver/index-inject` emit,
 * every subscriber pushes its current rows. Fresh per call, so subscribers
 * read live state (module graph, theme preference) at emit time.
 * @returns rows in subscriber activation order.
 */
collectIndexInjections(): IndexInjection[]

/**
 * Render one index.html body: the structured injection table first, then
 * the raw `tapIndex` transforms over the result.
 * @param html - the raw index.html body.
 * @returns the transformed body.
 */
renderIndex(html: string): string
```

Source: [`packages/host/webserver/src/index.ts`](../../packages/host/webserver/src/index.ts)

<a id="webserver-events"></a>

### `webserver/*` events

<a id="webserverindex-inject--emit"></a>

#### `webserver/index-inject` — emit

Collect the structured index injection table. Emitted on every index render and every worker boot-payload request; listeners push their current rows, so a row's data is read fresh at emit time.

```ts cordis-catalog
/**
 * Collect the structured index injection table. Emitted on every index
 * render and every worker boot-payload request; listeners push their
 * current rows, so a row's data is read fresh at emit time.
 * @param table - Mutable row table; listeners append in activation order.
 * @mode emit
 */
'webserver/index-inject'(table: IndexInjection[]): void
```

Source: [`packages/host/webserver/src/index.ts`](../../packages/host/webserver/src/index.ts)
<!-- END GENERATED cordis-surface -->
