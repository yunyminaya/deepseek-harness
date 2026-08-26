# Almacenamiento

[English](storage.md) | Español

El subsistema de almacenamiento persiste todo lo que no es un log de eventos de sesión (los logs de sesión tienen su propio seam — [persistence.md](persistence.es.md)). Es una capacidad opcional, no parte del tronco del agent loop (bucle del agente), dividida como un [seam de capacidad](../../.agents/notes/implemented/architecture/2026-06-13-capability-seams.es.md): el hub y la Service Definition ([dsh-storage](../../packages/storage/storage), `ctx.storage`), los Service Providers ([dsh-storage-json](../../packages/storage/storage-json), registrado como `json`, y [dsh-storage-sqlite](../../packages/storage/storage-sqlite), registrado como `sqlite`), y la forma de datos del Consumer ([dsh-storage-domain](../../packages/storage/storage-domain), `ctx.storageDomain`, también accesible como `ctx.storage.domain`) — el único Consumer del contrato de backend y la API tipada que usa todo lo demás. El hub no realiza IO por sí mismo: los backends poseen los medios, las formas de datos poseen la semántica, y los paquetes de producto nunca tocan los backends directamente. Registro de diseño: [Agent Note de almacenamiento KV por dominio](../../.agents/notes/proposed/architecture/2026-07-24-domain-kv-storage-and-workspace.es.md).

Fuente: [`packages/storage/storage/src/backend.ts`](../../packages/storage/storage/src/backend.ts) · [`packages/storage/storage-domain/src/spec.ts`](../../packages/storage/storage-domain/src/spec.ts) · [`packages/storage/storage-domain/src/events.ts`](../../packages/storage/storage-domain/src/events.ts)

## El hub: `ctx.storage`

`Storage` ([firmas](#ctxstorage--storage)) es un punto de encuentro, no un almacén. `ctx.storage.backend` es una tabla de nombre → backend: varios backends permanecen montados en paralelo, y qué backend sirve a qué consumer es configuración de ese consumer (la tabla de rutas de la capa de dominio), nunca una decisión global del hub. `register(name, backend)` devuelve el disposer; los nombres duplicados y las búsquedas desconocidas lanzan `StorageError`. La disposición solo desregistra el nombre — el plugin propietario cierra el backend después de desregistrarlo. Cada plugin de backend también publica una clave de servicio solo de ciclo de vida (`storageBackendServiceKey(name)`), que los providers de formas inyectan para que su activación no pueda competir con el registro del backend.

Las formas de datos se montan en el hub bajo un mapa de claves extensible por fusión:

```ts type-equiv
/**
 * Data forms mountable on the hub, keyed by form name. Form owners extend
 * this map via declaration merging (the domain layer merges
 * `domain: DomainFacility`) and mount the facility in their `apply`.
 */
interface StorageForms {}
```

`mount(form, facility)` es un efecto cuyo disposer desmonta; un segundo montaje de la misma clave lanza `duplicate-mount`. `form(form)` resuelve una facility montada y lanza `form-not-mounted` hasta que el plugin propietario carga — los ensamblajes ordenan los plugins en consecuencia en lugar de diferir en silencio. La capa de dominio fusiona `domain: DomainFacility`, de modo que `ctx.storage.domain` y `ctx.storageDomain` son el mismo objeto.

## El contrato de backend

```ts type-equiv
/**
 * One registered backend. A backend owns exactly one medium and shares its
 * lifecycle across all facets; facets are optional members — a backend that
 * cannot serve a data kind simply omits it, and resolution fails loud instead.
 */
interface StorageBackend {
  /** Key-value operations; absent when this backend cannot serve them. */
  readonly kv?: KvFacet

  /**
   * Drain in-flight writes across all open units and release the medium.
   * Idempotent; concurrent and repeated calls resolve once teardown finishes.
   * @returns resolution after the medium is released.
   */
  close(): Promise<void>
}
```

Un backend posee un medio (una raíz de árbol de archivos, un archivo de base de datos) y expone grupos de operaciones opcionales; hoy `kv` es el único grupo. `KvFacet.open(descriptor)` abre una unit con nombre — `KvUnitDescriptor` lleva el nombre, la versión del formato, los nombres de tabla y si existe un slot de singleton global — y devuelve un `KvUnit` con `loadAll`, `putRecord`, `deleteRecord`, `setGlobal` y `close`. Los nombres de unit y de tabla deben coincidir con `UNIT_NAME_RE` (seguros como nombre de archivo y como segmento de identificador SQL); las claves de registro son cadenas arbitrarias que nunca llegan a las rutas de archivo. Una unit no serializa las escrituras concurrentes — el orden le pertenece al llamador — pero cada llamada individual es atómica en el medio y durable una vez resuelta. Un medio sellado con una versión distinta rechaza con `version-mismatch`; uno que no puede parsearse como la unit rechaza con `malformed-medium` (sin migración, postura de pre-release). [`backend.ts`](../../packages/storage/storage/src/backend.ts) es el contrato normativo cláusula por cláusula, y la suite de conformidad compartida en [`tests/contract.ts`](../../packages/storage/storage/tests/contract.ts) comprueba cada cláusula contra cada backend. El [backend json](../../packages/storage/storage-json/README.es.md) republica atómicamente un archivo completo legible por humanos por unit; el [backend sqlite](../../packages/storage/storage-sqlite/README.es.md) almacena un documento por fila en una sola base de datos para datos que se actualizan con frecuencia.

## Declarar un dominio

Un dominio se declara una vez por su paquete propietario como un objeto spec — la fuente única de la identidad, la distribución y los schemas de registro del dominio (zod, de modo que `z.infer` mantiene los tipos de los consumers sin duplicar):

```ts type-equiv
/** Static declaration of one domain: identity, version, and record layout. */
interface DomainSpec {
  /** Domain name; must match `UNIT_NAME_RE` (doubles as the backend unit name). */
  readonly name: string
  /** Domain format version; a medium stamped with a different version rejects at open. */
  readonly version: number
  /** Optional global singleton slot. */
  readonly global?: DomainGlobalSpec<unknown>
  /** Table declarations keyed by table name; each name must match `UNIT_NAME_RE`. */
  readonly tables: Record<string, DomainTableSpec>
}
```

`defineDomain(spec)` fija los tipos literales del spec y falla de forma explícita en la carga del módulo del propietario, antes de tocar ningún medio: un nombre de dominio o de tabla fuera de `UNIT_NAME_RE`, una versión que no sea un entero no negativo, o un schema global que acepte `null` — todo eso lanza (`null` es el centinela de «nunca escrito» del medio, por lo que un global nullable almacenado no podría hacer round-trip). `domainTable<K, V>(schema)` declara una tabla con un tipo de clave phantom en tiempo de compilación (normalmente un [id branded](core.es.md#branded-ids)); `descriptorOf(spec)` proyecta el descriptor de unit orientado al backend.

## El dominio abierto

```ts type-equiv
/** One open domain, typed by its spec. */
interface Domain<S extends DomainSpec> {
  /** Domain name from the spec. */
  readonly name: string
  /** Global singleton handle; a spec without `global` has no usable handle (`never`). */
  readonly global: DomainGlobalHandleOf<S>
  /**
   * Resolve one declared table handle. Handles are stable — repeated calls
   * return the same instance.
   * @param name - Declared table name.
   * @returns the typed table handle.
   */
  table<N extends keyof S['tables'] & string>(name: N): KvTable<TableKeyOf<S, N>, TableValueOf<S, N>>

  /**
   * Close this domain: reject new writes immediately, drain already-queued
   * writes (their events still emit), release the backend unit, then free
   * the domain name for a later open. Idempotent — repeated calls share one
   * teardown. The consumer owns this call (typically as its own `ctx.effect`
   * disposer); the facility closes any domain left open when it unmounts.
   * @returns resolution after the unit is released.
   */
  close(): Promise<void>
}
```

Las lecturas son síncronas desde el estado en memoria autoritativo: `KvTable` expone `get`/`entries`/`keys`/`size` (iteradores de instantánea que permanecen estables mientras aterrizan las escrituras en cola), y el `get()` del identificador global sirve el `initial` del spec hasta que el primer `set` materializa el slot en el medio. Cada escritura — `put`, `delete`, `update`, `global.set` — se encola en una única cadena por dominio y primero alcanza la durabilidad del backend, luego muta la memoria y después emite `domain/changed`; una escritura de backend rechazada deja la memoria intacta, de modo que las lecturas nunca divergen del medio. `update(key, fn)` es un read-modify-write atómico en su slot de la cadena (una clave ausente rechaza con `missing-key`); el `delete` de una clave inexistente resuelve `false` sin escritura ni evento. Los registros devueltos son los propios objetos almacenados, no copias — reemplázalos con `put`/`update`, nunca los mutes in situ.

## La facility de dominio: `ctx.storageDomain`

`DomainFacility` ([firmas](#ctxstoragedomain--domainfacility)) abre los dominios declarados sobre backends enrutados. El enrutamiento es configuración del plugin de dominio, nunca del hub: `backend` nombra la ruta por defecto requerida y `routes` la anula por nombre de dominio. `open(spec)` ejecuta una secuencia estricta en la que cada paso hace fallar toda la llamada: rechaza un nombre ya abierto o que aún se está cerrando (`already-open`), resuelve la ruta (`backend-not-found`), exige la facet `kv` del backend (`facet-unsupported`), abre la unit (los `version-mismatch`/`malformed-medium` del backend pasan), y valida cada registro y global almacenado contra los schemas zod del spec (`invalid-record` con la tabla y la clave infractoras). El llamador posee el identificador devuelto y lo libera con `Domain.close()`; los dominios que siguen abiertos cuando el plugin se desmonta los cierra la facility, y el nombre de un dominio cerrado queda libre para reabrirse solo después de que el teardown complete del todo. `get(name)` es una búsqueda de diagnóstico sin tipar sobre el runtime `DomainImpl` privado del paquete que hay detrás de cada identificador tipado; `closeAll()` es la ruta de desmontaje.

## El evento de cambio: `domain/changed`

Cada escritura durable emite un evento estrictamente después de que el backend confirme la durabilidad, en el orden de la cadena de escritura del dominio ([entrada de evento](#domainchanged--emit)):

```ts type-equiv
/** Shared location fields of one durable domain change. */
interface DomainChangedBase {
  /** Owning domain name. */
  readonly domain: string
  /** Table name; `''` for a global-singleton write. */
  readonly table: string
  /** Record key; `''` for a global-singleton write. */
  readonly key: string
}
```

```ts type-equiv
/** One durable domain change; a closed union — switch on `operation`. */
type DomainChanged = DomainChangedPut | DomainChangedDeleted
```

`put` (inserciones, sobrescrituras y escrituras globales) lleva la nueva instantánea en `value` — nunca el valor anterior; un consumer que hace diffs conserva su propia instantánea previa. `deleted` es una tumba (tombstone) sin valor. El evento es una notificación, no un participante de transacción: en el momento de la emisión el punto de commit ya ha pasado, por lo que un listener que lanza sincrónicamente se contiene con una advertencia registrada en el log en lugar de rechazar la escritura ya durable, y los valores emitidos son iguales al estado en memoria en el momento de la emisión. El evento es solo intraproceso; el envío de cambios entre procesos es una limitación registrada ([README del paquete](../../packages/storage/storage-domain/README.es.md)).

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxstorage--storage"></a>

### `ctx.storage` — `Storage`

The storage hub service. Backends register under `backend`; data forms mount under their `StorageForms` key and are reached as `ctx.storage.<form>`.

```ts cordis-catalog
/**
 * Mount a data-form facility on the hub. Mounting is an effect: the
 * returned disposer unmounts the form.
 * @param form - Form key declared in {@link StorageForms}.
 * @param facility - The facility instance to expose.
 * @returns the disposer that unmounts the form.
 */
mount<K extends keyof StorageForms>(form: K, facility: StorageForms[K]): () => void

/**
 * Resolve a mounted data form.
 * @param form - Form key declared in {@link StorageForms}.
 * @returns the mounted facility.
 */
form<K extends keyof StorageForms>(form: K): StorageForms[K]
```

Source: [`packages/storage/storage/src/index.ts`](../../packages/storage/storage/src/index.ts)

<a id="ctxstoragedomain--domainfacility"></a>

### `ctx.storageDomain` — `DomainFacility`

The mounted domain facility. Opens declared domains over routed backends; one facility instance owns the open-domain table and enforces single-open per domain name.

```ts cordis-catalog
/**
 * Open one declared domain. Steps, each failing the whole call: reject a
 * name that is already open (`already-open`); resolve the backend route
 * (`backend-not-found` passes through from the hub); require its `kv` facet
 * (`facet-unsupported`); open the unit projected from the spec (backend
 * `version-mismatch`/`malformed-medium` pass through); load and validate
 * every stored record against the spec's zod schemas (`invalid-record`
 * with the offending table and key); construct the domain.
 *
 * Lifecycle: the CALLER owns the returned handle and closes it via
 * `Domain.close()` (typically as its own `ctx.effect` disposer) — the
 * facility does not tie the domain to any consumer fiber. Domains still
 * open when the facility unmounts are closed by the plugin disposer.
 * @param spec - The domain declaration, typically from `defineDomain`.
 * @returns the opened domain handle, typed by the spec.
 */
async open<S extends DomainSpec>(spec: S): Promise<Domain<S>>

/**
 * Look up an open domain by name, untyped. Diagnostic surface (the package
 * invariant cross-checks change events against live domain state); typed
 * consumers hold the handle returned by {@link open}.
 * @param name - Domain name.
 * @returns the open domain runtime, or `undefined` when not open.
 */
get(name: string): DomainImpl | undefined

/**
 * Close every domain still open on this facility. The unmount path for
 * consumers that never called `Domain.close()` themselves; closing is
 * idempotent, so double-closing an already-closed domain is harmless.
 * @returns resolution after every unit is released.
 */
async closeAll(): Promise<void>
```

Source: [`packages/storage/storage-domain/src/index.ts`](../../packages/storage/storage-domain/src/index.ts)

<a id="domain-events"></a>

### `domain/*` events

<a id="domainchanged--emit"></a>

#### `domain/changed` — emit

A domain record or the global singleton changed, emitted once per write strictly after the backend acknowledged durability. Events of one domain arrive in its write-chain order.

```ts cordis-catalog
/**
 * A domain record or the global singleton changed, emitted once per write
 * strictly after the backend acknowledged durability. Events of one
 * domain arrive in its write-chain order.
 * @param change - domain, table (`''` for global), key (`''` for global),
 * operation discriminant, and on `put` the new snapshot.
 * @mode emit
 */
'domain/changed'(change: DomainChanged): void
```

Source: [`packages/storage/storage-domain/src/events.ts`](../../packages/storage/storage-domain/src/events.ts)
<!-- END GENERATED cordis-surface -->
