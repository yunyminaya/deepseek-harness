# Módulos de cliente

[English](client-modules.md) | Español

La tabla de plugins web: la mitad Node del sistema de módulos de cliente de [dsh-client-modules](../../packages/client/modules), expuesta como `ctx.clientModules` (`ClientModuleRegistry`). Escanea las entradas del Loader del host en busca de paquetes que declaren `dsh.client`, compone el grafo de entradas `window.__DSH_BOOT__`, sirve cada bundle en `/plugins/<id>/client.js` y responde a cada colección de inyección de índice con las filas del manifest de arranque — las cuatro caras de un mismo servicio. Es una capacidad opcional de la pila de GUI web, no parte de la espina dorsal del agent loop, y es un consumer de [dsh-host-webserver](../../packages/host/webserver): el portador descrito en [web-server.md](web-server.es.md) suministra la ruta de prefijo y el evento `webserver/index-inject` al que responde este servicio. La mitad de navegador del mismo paquete (`ctx.modules`, la tabla de módulos lazy-CJS que obtiene y materializa estos bundles) es mecánica del kernel documentada en el [README del paquete](../../packages/client/modules/README.es.md), y no aquí.

Fuente: [`packages/client/modules/src/client/manifest.ts`](../../packages/client/modules/src/client/manifest.ts)

## El wire

El grafo es la fuente única de wire entre las mitades Node y navegador: el host compone las filas `WebBootEntry` a partir de los paquetes escaneados, publica el grafo como una fila de inyección `global` representada antes que las filas de script posteriores (`globalThis["__DSH_BOOT__"]`, con `<` escapado para que las cadenas controladas por plugins no puedan salirse del elemento script), y el shell lo analiza antes de arrancar nada. Una página sin un manifest válido no puede arrancar: el analizador del lado del navegador falla ruidosamente ante un grafo ausente o malformado.

```ts type-equiv
/**
 * One composed client entry pushed by the host (a graph row). Wire
 * single source: the host node half (package root) produces this same shape.
 * `immediately` marks stage-one prefetch; `inject` is informational graph
 * metadata (the authoritative edges live in each package's `dsh.client`
 * declaration and reach fibers through entry creation). `external` carries
 * module-graph edges: unlike `inject`, they constrain code arrival because
 * `require` is synchronous (see {@link WebBootGraph.entries}).
 */
interface WebBootEntry {
  /** Entry name == package name. */
  id: string
  /** Bundle endpoint, '/plugins/<id>/client.js?rev=<rev>'. */
  url: string
  /** Bundle content hash (cache-busting consistency anchor). */
  rev: string
  /** Package-name dependency edges, informational (preflight display / HMR diffing). */
  inject?: string[]
  /** Stage-one prefetch mark: load the script for factory registration during module-face boot. */
  immediately?: boolean
  /** Non-baseline module specifiers this row requests; omitted when it requests none. */
  external?: string[]
}
```

```ts type-equiv
/** The composed client entry graph the host injects as `window.__DSH_BOOT__`. */
interface WebBootGraph {
  /** Consistency anchor over the whole graph (content + bundle hashes). */
  rev: string
  /**
   * Composed entries in module-graph order — a dynamic package row precedes
   * rows whose `external` requests that package. Cordis activation order is
   * unrelated and remains owned by fiber service waiting.
   */
  entries: WebBootEntry[]
}
```

El `rev` de cada fila es el hash de contenido del bundle y viaja en la URL como consulta de invalidación de caché; el `rev` del grafo aplica hash a las filas compuestas, de modo que cualquier cambio de fila lo modifica. `immediately` marca el nivel de precarga de la etapa uno (obtener y ejecutar durante el arranque de la cara de módulos, solo registro); una fila lazy se obtiene en el primer import.

## El escaneo

Un paquete entra en la tabla declarando `dsh.client` (`platform: 'web'`, aristas `inject` opcionales, `immediately` opcional) en su package.json y exportando su bundle compilado en `exports["./client"]`. La resolución de paquetes se ancla en el `ctx.baseUrl` del árbol de configuración — el directorio cordis.yml, cuyo paquete declara cada plugin compuesto como dependencia — y la construcción lanza una excepción cuando ese ancla no está establecida.

El escaneo es incremental por paquete; no existe una ruta de código de reescaneo completo. Cada emisión cordis `internal/plugin` (construcción o disposición de fiber) marca como sucio el nombre de entrada de la fiber, y un flush de microtask concilia cada nombre sucio con las entradas vivas del loader. La pasada de activación siembra el mismo conjunto de sucios con todas las entradas actuales y hace flush de forma síncrona, de modo que el primer escaneo y el estado estacionario comparten una única implementación — con posturas de fallo opuestas. En la activación, una declaración malformada o un bundle ausente entre las entradas ya cargadas se agrega en un único `AggregateError` ruidoso que enumera cada paquete roto: la fiber FALLA y el barrido de fallo ruidoso del arranque lo informa. En estado estacionario, un paquete roto registra una advertencia y no debe contaminar a los demás.

Los metadatos del paquete — incluido el veredicto negativo de «no es un paquete de cliente» — se guardan en caché por nombre y nunca caducan: los cambios del conjunto de plugins surten efecto al reiniciar. Un reinicio de fiber reutiliza su fila y su rev sin tocarlos; los cambios de contenido del bundle llegan al grafo solo a través de `rebuilt()`.

## La ruta del bundle y la inyección de índice

`GET`/`HEAD /plugins/<id>/client.js` sirve el bundle registrado desde el disco con `no-cache` (la consulta rev, y no el caché HTTP, ancla la consistencia); los demás métodos devuelven 405. Un id desconocido — o una fila registrada cuyo bundle no se puede leer porque aún no se ha compilado — responde con un 404 ruidoso, de modo que ningún bundle ilegible aparece como una respuesta JavaScript exitosa. Las filas de inyección llevan el grafo actual en cada render del índice, de modo que una recarga siempre arranca contra la composición viva.

## El servicio

`ClientModuleRegistry` (`ctx.clientModules`, definido en [`packages/client/modules/src/index.ts`](../../packages/client/modules/src/index.ts)) expone las lecturas y la cara de reconstrucción; las firmas están en el [catálogo de servicio](#ctxclientmodules--clientmoduleregistry) generado. `graph()` devuelve el grafo compuesto actual (un objeto estable entre cambios) y `clientPath(id)` la ruta absoluta del bundle. `rebuilt(id)` es el único punto de entrada por el que el contenido del bundle llega al grafo: vuelve a aplicar hash al archivo, y solo un cambio real de rev recompone el grafo y notifica. `onRebuilt` se dispara por cada bundle cambiado con la nueva rev; `onGraphChanged` se dispara después de cualquier flush que haya recomponido el grafo (fila añadida o eliminada, o un cambio de rev por rebuilt) y es de modelo pull: los listeners vuelven a leer `graph()`. Ambas vías de notificación contienen las excepciones de los listeners, de modo que un suscriptor que lance una excepción no puede saltarse a los suscriptores posteriores ni matar lo que haya provocado el flush.

En desarrollo, [dsh-client-hmr](../../packages/client/hmr/README.es.md) es el driver de vigilancia del registro: su mitad Node hace stat-poll del bundle de cada fila del grafo desde una línea base capturada de forma síncrona, llama a `rebuilt(id)` ante un cambio, resincroniza su conjunto de vigilancia a través de `onGraphChanged` y difunde los cambios de rev a la mitad del navegador por SSE (Server-Sent Events). Los grafos de producción omiten por completo la fila de HMR (hot module replacement); el propio host de módulos nunca vigila archivos.

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.es.md).

<a id="ctxclientmodules--clientmoduleregistry"></a>

### `ctx.clientModules` — `ClientModuleRegistry`

The web plugin table service: incremental `dsh.client` scan + wire composition + bundle route + index injection rows. Construction runs the activation scan synchronously — a malformed declaration or missing bundle among the already-loaded entries aggregates into one loud throw (FAILED fiber; the boot activation audit reports it).

```ts cordis-catalog
/**
 * Current composed entry graph (stable object between changes).
 * @returns the graph served as `window.__DSH_BOOT__`.
 */
graph(): WebBootGraph

/**
 * Absolute path of an entry's client bundle.
 * @param id - entry id (package name).
 * @returns the path, or undefined for an unknown id.
 */
clientPath(id: string): string | undefined

/**
 * Re-hash one bundle (the HMR watch's registration hook — the only entry
 * point through which bundle content changes reach the graph).
 * @param id - entry id (package name).
 * @returns the new rev, or undefined for an unknown id.
 */
rebuilt(id: string): string | undefined

/**
 * Subscribe to bundle rebuilds; fires only when the re-hash changed the rev.
 * @param listener - receives the entry id and its new bundle rev.
 * @returns the unsubscriber.
 */
onRebuilt(listener: (id: string, rev: string) => void): () => void

/**
 * Fires after any flush that recomposed the graph (row added/removed, or a
 * rebuilt rev change). Pull model: listeners re-read {@link graph}.
 * @param listener - notified with no payload.
 * @returns the unsubscriber.
 */
onGraphChanged(listener: () => void): () => void
```

Source: [`packages/client/modules/src/index.ts`](../../packages/client/modules/src/index.ts)
<!-- END GENERATED cordis-surface -->
