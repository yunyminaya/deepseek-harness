# Invariantes de ejecución

[English](invariants.md) | Español

[dsh-invariants](../../packages/runtime-diagnostics/invariants) es el servicio de registro configurable (`ctx.invariants`) para las comprobaciones de invariantes de ejecución propias de cada paquete. Es un paquete del grupo de soporte, no un seam de capacidad de tres paquetes, y no forma parte del núcleo del agent loop (bucle del agente): el registro es dueño de la selección, la reserva de nombres, el ciclo de vida de las fibras hijas y los fallos atribuidos al paquete, mientras que cada paquete del workspace publica un plugin compañero `./invariant` que registra comprobaciones bajo su nombre npm exacto. Lo que una comprobación puede afirmar — flujos de eventos autoritativos o datos mutables, nunca la presencia de un servicio o método — es la convención de invariantes de ejecución de [AGENTS.md](../../AGENTS.md#conventions); el diseño del registro lo posee el [Agent Note del servicio de invariantes](../../.agents/notes/implemented/architecture/2026-07-19-package-owned-invariant-service.es.md).

Código fuente: [`packages/runtime-diagnostics/invariants/src/index.ts`](../../packages/runtime-diagnostics/invariants/src/index.ts)

## Selección

```ts type-equiv
/** Runtime invariant selection configured on the service plugin. */
interface Config {
  /** Global switch; defaults to `true`. */
  readonly enabled?: boolean
  /** Case-sensitive JavaScript regex sources that admit package names; empty admits all. */
  readonly package_allowlist?: string[]
  /** Case-sensitive JavaScript regex sources that exclude package names after allowlist matching. */
  readonly package_blocklist?: string[]
}
```

Un paquete se selecciona cuando el servicio está habilitado, la lista de permitidos está vacía o al menos un patrón coincide con su nombre npm completo, y ningún patrón de la lista de bloqueados coincide — una coincidencia de bloqueo anula una coincidencia de permitido. Las entradas se compilan con `new RegExp(source)`: la coincidencia no está anclada salvo que el origen aporte `^` y `$`, y la sintaxis `/pattern/flags` no se analiza. La validación falla ruidosamente al arrancar el servicio: una entrada vacía, con espacios alrededor, duplicada o inválida lanza una excepción en lugar de omitirse. Un patrón válido puede no coincidir con ningún paquete cargado actualmente, así que la carga posterior y el HMR (hot module replacement) siguen siendo deterministas; los filtros quedan fijos durante toda la vida del servicio ([README](../../packages/runtime-diagnostics/invariants/README.es.md)).

## El instalador

```ts type-equiv
/**
 * Throw a package-attributed invariant failure.
 * @param message - violated package contract without the standard prefix.
 * @returns never because reporting a violation throws.
 */
type InvariantFailure = (message: string) => never
```

```ts type-equiv
/** Install one package's checks into the registration's child context. */
interface InvariantInstaller {
  /**
   * Install the package contribution.
   * @param ctx - child context owned by this invariant registration.
   * @param fail - reporter bound to the registering package name.
   * @returns nothing, or a promise settling after asynchronous checks finish.
   */
  (ctx: Context, fail: InvariantFailure): void | Promise<void>
  /** Services the child installer fiber may access. */
  readonly inject?: Inject
}
```

Un instalador habilitado se ejecuta en una fibra hija Cordis dedicada; `installer.inject` declara los servicios a los que esa fibra puede acceder, y se espera la finalización del instalador, síncrona o asíncrona, antes de que el registro tenga éxito. `fail(message)` lanza `InvariantError` — `extends Error` con un `code: 'INVARIANT'` estable, el `packageName` propietario y un mensaje con el prefijo `invariant violated by "<package>": …` — de modo que una violación es atribuible sin que el registro importe ningún paquete de producto.

## El servicio

`ctx.invariants.register(packageName, installer)` reserva un registro activo para el nombre npm completo y devuelve su disposer con ámbito de efecto. La reserva se mantiene incluso cuando los filtros dejan inactivo al instalador, así que dos plugins nunca pueden reclamar en silencio el mismo nombre de paquete; un nombre duplicado, vacío o con espacios lanza una excepción. Un fallo del instalador dispone la fibra hija y libera la reserva atómicamente. El servicio es dueño de cada fibra de registro, mientras que el disposer devuelto también pertenece a la fibra compañera: descargar cualquiera de los dos lados elimina los listeners, el estado de trazado y la reserva, de modo que un compañero puede recargarse y registrar de nuevo el mismo nombre sin estado retenido.

## El contrato del compañero

Cada paquete del workspace posee un compañero `./invariant` ([contrato de paquete](../../packages/AGENTS.md)); la publicación y el registro son exhaustivos, pero las aserciones no son deliberadamente sintéticas. Un compañero instala una comprobación solo cuando su paquete posee una relación observable de eventos o de datos mutables; si no, exporta un instalador vacío cuyo comentario inicial empieza por `No runtime invariant:` y explica, de forma específica para el paquete, por qué no hay nada comprobable. `pnpm run verify-package-invariants` rechaza mecánicamente los marcadores generados, los instaladores vacíos sin explicación, los instaladores no vacíos que omiten o ignoran el reporter, los nombres de registro incorrectos y el cableado incompleto de exportación, publicación, dependencias o bundle ([Agent Note de reglas mecánicas](../../.agents/notes/implemented/architecture/2026-07-19-package-invariant-runtime-contracts.es.md)). El catálogo de compañeros ejecutables y la composición estándar viven en el [README del paquete](../../packages/runtime-diagnostics/invariants/README.es.md).

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxinvariants--invariantregistry"></a>

### `ctx.invariants` — `InvariantRegistry`

Package-owned invariant registry with global and regex-based selection.

```ts cordis-catalog
/**
 * Register one package's invariant installer. The package name is reserved
 * even when filtering disables its checks. Enabled installers run in a child
 * fiber; failure disposes that fiber and releases the reservation.
 * @param packageName - full npm package name that owns the contribution.
 * @param installer - listener or startup-check installer for the child context.
 * @returns an effect-scoped disposer for the registration.
 */
register(packageName: string, installer: InvariantInstaller): () => void
```

Source: [`packages/runtime-diagnostics/invariants/src/index.ts`](../../packages/runtime-diagnostics/invariants/src/index.ts)
<!-- END GENERATED cordis-surface -->
