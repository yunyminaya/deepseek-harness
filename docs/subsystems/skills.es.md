# Skills

[English](skills.md) | Español

La [familia de capacidades de skill (destreza)](../../packages/skill) incluye la Service Definition ([dsh-skill](../../packages/skill/skill), `ctx.skills`), el Service Provider local ([dsh-skill-filesystem](../../packages/skill/skill-filesystem)), el provider de insignias empaquetado opcional ([dsh-skill-badge](../../packages/skill/skill-badge)) y el Consumer ([dsh-tool-skill](../../packages/skill/tool-skill)). El registro fusiona los catálogos de los providers entre su capa de host y sus capas por scope; los providers aportan skills locales o empaquetados; el Consumer posee los catálogos inicial y de reemplazo, además de la herramienta `skill` orientada al modelo. Los skills son instrucciones opcionales, no eventos de sesión, por lo que su vocabulario vive aquí y no en [core.md](core.es.md).

Fuente: [`packages/skill/skill/src/index.ts`](../../packages/skill/skill/src/index.ts), [`packages/skill/skill-filesystem/src/index.ts`](../../packages/skill/skill-filesystem/src/index.ts), [`packages/skill/skill-badge/src/index.ts`](../../packages/skill/skill-badge/src/index.ts) y [`packages/skill/tool-skill/src/index.ts`](../../packages/skill/tool-skill/src/index.ts).

## Registro de providers

`ctx.skills` combina providers locales, integrados, remotos u otros. El registro es síncrono; la inicialización y el descubrimiento remotos pertenecen al `list()` en espera. Los objetos, las opciones y los candidatos de los providers se toman prestados en modo readonly, mientras que los campos semánticos se validan.

El registro está en capas host+por-scope, la forma que el [registro de herramientas](tools.es.md) estableció sobre [dsh-scope](../../packages/core/scope): un registro se archiva en la capa del scope del contexto que lo llama, de modo que las filas de host y los plugins del repositorio aterrizan en la capa global, mientras que un plugin montado por la composición permanente del preset de un agent (agente) aterriza en la capa de ese preset; los nombres de los providers son únicos por capa, no a nivel de proceso. Una lectura fusiona la capa global con la cadena del scope que observa: la entrada de la capa más cercana gana de forma directa un nombre de skill duplicado, y el orden de rango que se indica abajo decide los duplicados solo dentro de una misma capa. Las cachés de descubrimiento se indexan por la cadena de scope resuelta, de modo que re-parentar un scope (una recomposición de sesión en blanco) es visible en la siguiente lectura sin necesidad de mutar el registro.

Dentro de una capa, los nombres duplicados se resuelven por rango, luego por orden del provider y después por orden local; los resúmenes se ordenan por nombre. Un `list()` rechazado se registra en el log y se omite de una observación incompleta, mientras que una observación incompleta explícita aporta candidatos utilizables sin que el resultado sea cacheable; los candidatos malformados fallan rápido. Cada factory de provider recibe un control con alcance de registro cuyo `invalidate()` limpia los catálogos completados solo mientras ese registro exacto siga activo, y cuya señal aborta ante un registro fallido o una disposición. Un descubrimiento en curso se reintenta una vez cuando cambia la generación de su provider; un segundo cambio devuelve los candidatos más recientes como incompletos y sin cachear. Las mutaciones de providers y de runtime emiten el evento de invalidación sin filtrar `skills/change`; no lleva diff, por lo que los consumers vuelven a obtener `snapshot()` con sus propias opciones de búsqueda.

Un array devuelto por `SkillProvider.list()` es una abreviatura de descubrimiento completo. `SkillProviderObservation` permite a un provider exponer candidatos que siguen siendo cargables directamente, a la vez que informa de que la observación no es autoritativa.

```ts type-equiv
/** Provider candidates plus whether the current discovery is authoritative. */
interface SkillProviderObservation {
  /** Candidates available from the current provider discovery. */
  readonly candidates: readonly SkillCandidate[]
  /** Whether discovery completed and these candidates may be cached. */
  readonly complete: boolean
}
```

```ts type-equiv
/** Provider interface for one source of skills, such as local directories or a remote registry. */
interface SkillProvider {
  /** Unique provider name in the `ctx.skills` registry. */
  readonly name: string
  /**
   * List available skill candidates for the current lookup context. Provider
   * plugins register synchronously during `apply()`; remote initialization,
   * authentication, and discovery are awaited inside this method. Implementations
   * should settle promptly when `options.signal` aborts.
   * @param options - lookup options; `cwd` selects workspace-sensitive skills and `signal` cancels work.
   * @returns provider candidates as a complete-array shorthand, or an explicit
   *   observation when usable candidates came from incomplete discovery.
   */
  readonly list: (options: SkillLookupOptions) => Promise<readonly SkillCandidate[] | SkillProviderObservation>
  /**
   * Load a complete skill body for a previously listed candidate.
   * @param candidate - the winning candidate originally returned by this provider.
   * @param options - lookup options; `cwd` selects workspace-sensitive skills and `signal` cancels work.
   * @returns the full skill body, or `undefined` if it is no longer loadable.
   */
  readonly get: (candidate: SkillCandidate, options: SkillLookupOptions) => Promise<SkillDefinition | undefined>
}
```

```ts type-equiv
/** Registration-scoped lifecycle and invalidation capability borrowed by one provider. */
interface SkillProviderControl {
  /** Aborts if registration fails or when the exact provider registration is disposed. */
  readonly signal: AbortSignal
  /** Invalidate completed catalogs and notify consumers only while the exact registration remains active. */
  readonly invalidate: () => void
}
```

## Prioridad del descubrimiento local

El provider local incluido explora las raíces en orden de rango:

| Rango | Fuente | Raíz |
|---|---|---|
| 100 | `project-dsh` | `<projectRoot>/.dsh/skills` |
| 200 | `project-agents` | `<projectRoot>/.agents/skills` |
| 300 | `custom` | `Config.customSkillDirs` |
| 400 | `user-dsh` | `<dshHome>/skills` |
| 500 | `user-agents` | `<agentsHome>/skills` |
| 600 | `bundled` | `Config.bundledSkillDir` when configured |

La raíz del proyecto es el ancestro más cercano que contiene `.git`; si no hay ninguno, se usa el cwd actual. Cuando `ctx.fs` está disponible, el recorrido hacia la raíz de git comprueba `.git` a través del servicio de sistema de archivos, de modo que los workspaces remotos o en sandbox no caen en el límite del sistema de archivos del host. La raíz DSH de usuario omite su hijo `.system`. El provider local no sintetiza skills de sistema integrados; los despliegues suministran skills empaquetados a través de raíces bundled configuradas o de providers dedicados.

`dsh-skill-badge` registra un candidato `bundled` inmutable en `BUNDLED_SKILL_RANK` y expone su directorio de assets empaquetados a través de `resourceBase`. El CLI incluido declara el plugin deshabilitado, de modo que habilitar su fila de composición es una aceptación explícita.

Chokidar observa las raíces existentes para detectar altas y bajas directas de bundle/entradas planas, además de cambios directos en entradas de skill. Una raíz inexistente se sigue segmento de ruta ausente a la vez desde su ancestro existente más cercano hasta que Chokidar puede engancharse. Los archivos de recursos bajo un bundle no son cambios de catálogo. Las observaciones `write` y `edit` orientadas al modelo invalidan sincrónicamente el provider cuando su objetivo es relevante para el catálogo, mientras que el watcher del host cubre las mutaciones de IDE, Git, shell y procesos externos. Los fallos del watcher dejan la observación actual incompleta sin ocultar los candidatos legibles a las cargas directas; los watchers con alcance de proyecto usan un LRU acotado configurado.

## Identidad de los skills

Los nombres de skill usan kebab-case (`^[a-z0-9]+(?:-[a-z0-9]+)*$`). El provider local acepta bundles de directorio (`<name>/SKILL.md`) y archivos Markdown planos (`<name>.md`). No se admite el descubrimiento recursivo anidado de `**/SKILL.md`.

```ts type-equiv
/** Origin bucket for a skill contribution. The value is prompt-visible metadata, not precedence by itself. */
type SkillSource = 'project-dsh' | 'project-agents' | 'runtime' | 'user-dsh' | 'user-agents' | 'custom' | 'bundled' | (string & {})
```

## Resúmenes, candidatos y definiciones completas

`SkillSummary` es la forma de resumen neutral respecto a la invocación del registro. Los consumers eligen qué entradas y campos renderizar; el catálogo de sesión del modelo usa solo `name` y `description` invocables por el modelo, nunca el cuerpo ni la ruta de archivo absoluta. `SkillInvocationPolicy` normaliza los dos controles de invocación independientes en booleanos positivos, y todo resumen, candidato y definición resuelto lo lleva consigo sin convertir frontmatter arbitrario en el modelo de dominio.

```ts type-equiv
/** Invocation controls shared by skill discovery consumers. */
interface SkillInvocationPolicy {
  /** Whether model-facing catalogs and loaders include this skill. */
  readonly modelInvocable: boolean
  /** Whether human-facing command catalogs and loaders include this skill. */
  readonly userInvocable: boolean
}
```

```ts type-equiv
/** Invocation-neutral skill metadata returned by `ctx.skills.list()`. */
interface SkillSummary {
  /** Kebab-case identifier used to address the skill. */
  readonly name: string
  /** Short routing description shown by discovery consumers. */
  readonly description: string
  /** Optional extra routing guidance. */
  readonly whenToUse?: string
  /** Resolved model and user invocation controls. */
  readonly invocation: SkillInvocationPolicy
  /** Discovery source that produced this winning skill. */
  readonly source: SkillSource
  /** Provider that owns this skill body. */
  readonly provider: string
  /** Provider-specific base for relative resources. */
  readonly resourceBase?: SkillResourceBase
}
```

`ctx.skills.list()` conserva las cuatro combinaciones de política. `isModelInvocable(skill)` y `isUserInvocable(skill)` leen el campo obligatorio correspondiente. Un skill solo para el modelo establece `{ modelInvocable: true, userInvocable: false }`, un skill solo para el usuario establece `{ modelInvocable: false, userInvocable: true }`, y establecer ambos campos en `false` mantiene el skill disponible solo a través de llamadores de confianza de `ctx.skills.get()`. El provider local lee las claves exactas de frontmatter en kebab-case `disable-model-invocation` y `user-invocable`, asigna `true` por defecto a los campos omitidos y proyecta cada skill parseado en esta política normalizada.

`SkillCatalogSnapshot` distingue la ausencia autoritativa del fallo transitorio de un provider o de un catálogo que siguió cambiando durante el descubrimiento. `skills` contiene los resúmenes ordenados y neutrales respecto a la invocación recopilados en esa observación; `complete` solo es true cuando todos los providers registrados completaron sin una revisión concurrente del catálogo. Las instantáneas incompletas no se cachean, lo que permite a cada consumer conservar su último catálogo filtrado bueno y reintentar.

```ts type-equiv
/** One catalog observation plus whether discovery completed within a stable catalog revision. */
interface SkillCatalogSnapshot {
  /** Sorted invocation-neutral summaries collected in this observation. */
  readonly skills: SkillSummary[]
  /** Whether every registered provider completed without a concurrent catalog revision. */
  readonly complete: boolean
}
```

`SkillCandidate` es la forma de provider a registro. `locator` es estado opaco del provider; el registro solo lo almacena y se lo devuelve al `get()` del provider ganador.

```ts type-equiv
/** Provider catalog entry used by the registry to merge and later load skills. */
interface SkillCandidate extends SkillSummary {
  /** Lower ranks win duplicate skill names before provider registration order is considered. */
  readonly rank: number
  /** Opaque provider-owned handle passed back to `provider.get()`. */
  readonly locator: unknown
  /** Absolute file path when the provider has one. */
  readonly path?: string
  /** Parsed optional metadata object from provider-specific skill frontmatter. */
  readonly metadata?: Readonly<Record<string, unknown>>
}
```

`SkillDefinition` es el resultado parseado completo que devuelve `ctx.skills.get()` y que usa la herramienta `skill`. `resourceBase` le dice a la herramienta cómo renderizar la guía de recursos relativos para skills locales, por URL o gestionados por un provider.

```ts type-equiv
/** Optional provider-specific base used by loaded skill bodies to resolve relative resources. */
type SkillResourceBase =
  | { readonly kind: 'directory'; readonly path: string }
  | { readonly kind: 'url'; readonly url: string }
  | { readonly kind: 'opaque'; readonly description: string }
```

```ts type-equiv
/** Complete parsed skill definition, including the body loaded by `ctx.skills.get()`. */
interface SkillDefinition extends SkillSummary {
  /** Markdown instruction body after any provider-specific metadata removal. */
  readonly content: string
  /** Absolute file path when the skill came from disk. */
  readonly path?: string
  /** Parsed optional metadata object from frontmatter. */
  readonly metadata?: Readonly<Record<string, unknown>>
}
```

Las entradas de skill en runtime pueden omitir los controles de invocación y la etiqueta del provider. El registro resuelve ambos valores por defecto una sola vez y luego usa la misma forma de definición completa y el mismo orden de recolección primero-gana que los providers. El disposer devuelto elimina la contribución e invalida las cachés de descubrimiento.

```ts type-equiv
/** Runtime skill contribution accepted by `ctx.skills.register()`. */
type SkillRegistration = Omit<SkillDefinition, 'invocation' | 'provider'> & {
  /** Invocation controls; omission permits both model and user surfaces. */
  readonly invocation?: SkillInvocationPolicy
  /** Provider label; omission uses the registry-owned runtime provider. */
  readonly provider?: string
}
```

## Búsqueda y configuración

La búsqueda de skills es sensible al cwd porque los providers pueden exponer skills locales del workspace, y su señal opcional cancela el trabajo del provider para el llamador. Las lecturas del registro además reciben el scope que observa — los consumers pasan el agent llamador, que es su propia clave de scope — a través de `SkillViewOptions`; el registro consume `scope` para seleccionar capas, y los providers leen solo su contrato `SkillLookupOptions` del mismo objeto de opciones prestado. La cancelación se comprueba antes y después de la selección del catálogo, incluidos los aciertos de caché, y compite con el descubrimiento y con la carga de definiciones completas. Si no se encuentra ninguna raíz de git, el provider local trata el cwd suministrado como la raíz del proyecto.

El registro no cachea las definiciones completas. Cada `get()` llama al provider ganador con el candidato seleccionado, de modo que el provider local vuelve a leer el cuerpo actual. Una definición cuyo nombre ya no coincide con ese candidato se rechaza e invalida al provider exacto para un nuevo descubrimiento.

```ts type-equiv
/** Caller context used for cwd-sensitive and abortable provider work. */
interface SkillLookupOptions {
  /** Workspace selector for the current lookup. */
  readonly cwd?: string | undefined
  /** Abort discovery or loading work for the current caller. */
  readonly signal?: AbortSignal | undefined
}
```

```ts type-equiv
/**
 * Registry read options: provider lookup context plus the viewing scope.
 * The registry consumes `scope` to select layers; providers receive the same
 * borrowed options object and read only their {@link SkillLookupOptions}
 * contract from it.
 */
interface SkillViewOptions extends SkillLookupOptions {
  /** Viewing scope (the calling agent); omitted reads the global layer alone. */
  readonly scope?: ScopeKey | undefined
}
```

El registro posee solo su límite de caché de descubrimiento. El provider local posee las raíces del sistema de archivos (`dshHome`, `agentsHome`, `customSkillDirs` y los opcionales `bundledSkillDir`/`DSH_BUNDLED_SKILL_DIR`), además de los controles de habilitación del watcher, sondeo, estabilidad, symlink y capacidad del proyecto. El consumer posee su límite de descripción de catálogo. Los valores por defecto exactos y la validación están en el [catálogo de configuración](../config-catalog.es.md) generado.

```ts type-equiv
/** Skill registry configuration. */
interface Config {
  /** Maximum number of completed cwd/provider catalogs kept in memory. */
  readonly collectCacheMaxEntries?: number
}
```

## Catálogo de sesión y contrato de la herramienta

`dsh-tool-skill` inyecta el `<system-reminder>` durable inicial con rol de usuario en el primer `agent/pre-step` de una sesión en vivo que observa una vista completa no vacía. El catálogo contiene solo el `name` de skill ordenado y la `description` normalizada y escapada para XML; omite cuerpos, rutas, fuentes, providers y pistas de enrutamiento. El descubrimiento reenvía la señal de aborto del paso a través de `SkillLookupOptions`. `catalogDescriptionMaxLength` es la config del consumer para el límite de descripción, con valor por defecto `500` y mínimo entero `3`.

Antes de cada paso de modelo posterior, el consumer aplica la visibilidad exacta de herramientas y calcula el digest de las entradas renderizadas exactas entre las etiquetas `<available_skills>` de una instantánea completa. Deriva la línea base de comparación de las mismas entradas del mensaje de catálogo visible reconocible más reciente originado por el plugin. Un digest cambiado añade un reemplazo completo durable a través de `agent.inject()`; eliminar todos los skills añade un reemplazo vacío explícito. Las instantáneas incompletas conservan la última vista buena del modelo. Si la compactación oculta todos los mensajes de catálogo históricos, la siguiente instantánea completa restablece el catálogo actual; una vista vacía sin catálogo previo no emite nada. Estos mensajes de catálogo son historial de sesión, no World State.

La herramienta `skill({ name })` orientada al modelo valida el nombre en kebab-case, encuentra el resumen en el catálogo neutral respecto a la invocación, lo rechaza antes de cargarlo salvo que `isModelInvocable` permita el acceso, y luego vuelve a leer la definición completa para el cwd del agent llamador y vuelve a comprobar la política antes de devolver el contenido. Informa de un skill no resuelto como desconocido o ya no disponible, y devuelve un resultado de herramienta que contiene `<skill_content name="...">`, `<skill_resources>` y `<skill_instructions>`. `resourceBase` resuelve solo cuando hace falta los scripts, las referencias y los assets referenciados explícitamente; el resultado cargado no enumera el directorio del skill. Por tanto, las ediciones solo del cuerpo cambian las llamadas de herramienta posteriores sin producir mensajes de catálogo ni reescribir resultados de herramienta anteriores.

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxskills--skillregistry"></a>

### `ctx.skills` — `SkillRegistry`

Layered registry of skill providers, the host+per-scope shape the tools registry established. A registration files into the layer of its calling context's scope (scopeOf): host rows and repository plugins land in the global layer, while a plugin mounted by an agent preset's standing composition lands in that preset's layer. A read merges the global layer with the viewing scope's chain — the nearest layer's entry wins a duplicate name outright, and the rank order decides duplicates only within one layer. It exposes sorted invocation-neutral summaries and loads full skill bodies on demand.

```ts cordis-catalog
/**
 * Register a borrowed same-process provider synchronously during plugin
 * apply, into the calling context's layer: a scoped context (an agent
 * preset's standing mount) registers for that scope alone, an unscoped
 * context registers globally. Duplicate names within one layer and reserved
 * names throw; remote initialization belongs in `list()`. Fiber disposal
 * unregisters the provider and invalidates catalog caches.
 * @param create - synchronous factory receiving this registration's lifecycle and invalidation control.
 * @returns the exact Cordis effect disposer that unregisters this provider;
 *   composite effects may yield it directly to preserve teardown ordering.
 */
registerProvider(create: (control: SkillProviderControl) => SkillProvider): () => void

/**
 * Register a borrowed readonly runtime skill into the calling context's
 * layer. Project entries outrank runtime entries, which outrank user
 * entries, within one layer. Same-name runtime entries in one layer are
 * first-wins; a duplicate logs a warning and receives a no-op disposer so
 * it cannot remove the winner.
 * @param skill - the skill definition input; omitted invocation and provider fields receive defaults.
 * @returns the exact Cordis effect disposer, preserving composite teardown order and invalidating caches.
 */
register(skill: SkillRegistration): () => void

/**
 * List invocation-neutral skill summaries for a workspace. Consumers apply
 * model or user invocation policy at their operational boundary. Lookup
 * options and provider candidates are readonly same-process values borrowed
 * throughout discovery.
 * @param options - view options; `scope` selects the viewing agent's layers, `cwd` selects project roots, and `signal` cancels discovery.
 * @returns all sorted winning summaries.
 */
async list(options: SkillViewOptions = {}): Promise<SkillSummary[]>

/**
 * Observe the current invocation-neutral catalog and whether discovery completed within a stable revision.
 * Incomplete observations are never cached, allowing consumers to retain last-good state and
 * retry on their next request boundary.
 * @param options - view options; `scope` selects the viewing agent's layers, `cwd` selects project roots, and `signal` cancels discovery.
 * @returns sorted summaries plus discovery-completeness state.
 */
async snapshot(options: SkillViewOptions = {}): Promise<SkillCatalogSnapshot>

/**
 * Load and validate the winning candidate, passing its opaque discovery locator back to the
 * provider. Cancellation is rechecked after selection, including cache hits, and raced against
 * loading so an uncooperative provider cannot hang the caller.
 * @param name - kebab-case skill name.
 * @param options - view options; `scope` selects the viewing agent's layers,
 *   `cwd` selects workspace-sensitive skills, and `signal` cancels work.
 * @returns the full skill, including body content, or `undefined`.
 */
async get(name: string, options: SkillViewOptions = {}): Promise<SkillDefinition | undefined>
```

Source: [`packages/skill/skill/src/index.ts`](../../packages/skill/skill/src/index.ts)

<a id="skills-events"></a>

### `skills/*` events

<a id="skillschange--emit"></a>

#### `skills/change` — emit

A skill provider, runtime contribution, or provider-backed catalog may have changed. This is an unfiltered invalidation notification; consumers refetch the catalog for their own lookup options. Listener failures are contained and cannot veto the registry mutation.

```ts cordis-catalog
/**
 * A skill provider, runtime contribution, or provider-backed catalog may
 * have changed. This is an unfiltered invalidation notification; consumers
 * refetch the catalog for their own lookup options. Listener failures are
 * contained and cannot veto the registry mutation.
 * @mode emit
 */
'skills/change'(): void
```

Source: [`packages/skill/skill/src/index.ts`](../../packages/skill/skill/src/index.ts)
<!-- END GENERATED cordis-surface -->
