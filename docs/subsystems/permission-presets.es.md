# Presets de permisos

[English](permission-presets.md) | Español

La capa de presets de permisos de [dsh-permission-presets](../../packages/interaction/permission-presets) (`ctx.permissionPresets`, `PermissionPresetService`) agrupa los dos knobs de aplicación independientes — el [modo de sandbox](sandbox.es.md) (`sandbox/mode`) y la [política de aprobación](approval.es.md) (`approval/policy`) — en presets con nombre que un cliente ofrece como un único selector de Permissions. Es una capacidad opcional, no forma parte de la columna vertebral del agent loop, y no posee ninguna aplicación: la ejecución, la narración de prompts y el replay siguen leyendo los pliegues de sus knobs, y un cambio de preset solo registra la intención y escribe a través del setter canónico de cada knob. El [README del paquete](../../packages/interaction/permission-presets/README.es.md) es el responsable del estado de composición y de las limitaciones; el [diseño de cambio de sandbox](../../.agents/notes/implemented/feature/2026-07-06-sandbox.es.md) es el responsable de la justificación.

Fuente: [`packages/interaction/permission-presets/src/index.ts`](../../packages/interaction/permission-presets/src/index.ts)

## La tabla de presets

Un preset es una clave de tabla que mapea a un bundle de sandbox/aprobación más la presentación opcional del cliente; la tabla por defecto incluye `workspace-write` (`workspace-write` + `ask`) y `danger-full-access` (`danger-full-access` + `never`).

```ts type-equiv
/** One preset's sandbox/approval bundle and optional client presentation. */
interface PresetSpec {
  /** The `sandbox/mode` value the preset writes through. */
  sandbox: SandboxMode
  /** The `approval/policy` value the preset writes through. */
  approval: ApprovalPolicy
  /** The display label a client shows for this preset; the raw table key when omitted. */
  name?: string
  /** One user-facing sentence on what the preset means; omitted when not configured. */
  description?: string
}
```

```ts type-equiv
/** The {@link PermissionPresetService} config: preset table and composition default. */
interface Config {
  /**
   * The preset table: name → knob bundle. Defaults to `workspace-write`
   * (workspace-write + ask) and `danger-full-access` (danger-full-access +
   * never). The name `custom` is reserved for the derived not-a-preset state.
   */
  presets?: Record<string, PresetSpec>
  /**
   * Default for new sessions. When omitted, the preset matching the composed
   * sandbox and approval defaults is used.
   */
  defaultPreset?: string
}
```

El servicio requiere un ejecutor `ctx.shell` confinante y `ctx.approval`, y una mala configuración falla al cargar el plugin: una entrada de tabla llamada `custom` lanza una excepción (el nombre está reservado para el estado derivado de no-preset), y componer sobre un ejecutor bash que no confina (sin el hecho de capacidad `sandboxMode`) lanza una excepción, porque los presets agrupan un modo de sandbox.

## El preset actual y el `custom` derivado

`current(events)` deriva el preset efectivo de los knobs, no solo de su propio evento: pliega el modo de sandbox efectivo de la sesión (con respaldo al modo configurado del ejecutor) y la política de aprobación efectiva (con respaldo al config del servicio de aprobación y luego a `ask`), prefiere una selección registrada que siga coincidiendo, después la primera entrada de tabla que coincida en orden de declaración y, en caso contrario, devuelve `CUSTOM_PRESET` (`'custom'`). `custom` es solo derivado: los clientes pueden mostrarlo como valor actual, pero nunca es un objetivo de cambio ni una carga de evento.

`names` lista los presets conmutables en orden de declaración de la tabla; `optionOf(name)` construye la opción que un cliente renderiza para una clave de tabla (la etiqueta usa la clave como respaldo) o para `custom`, y lanza una excepción para cualquier otro nombre.

```ts type-equiv
/** The select-option shape a presentation layer advertises for one preset (or for the derived `custom` state). */
interface PresetOption {
  /** Stable option value: the table key, or `custom`. */
  value: string
  /** The display label. */
  name: string
  /** One user-facing sentence on what the value means; omitted when not configured. */
  description?: string
}
```

## El cambio y el evento `permission/preset`

`set(session, name)` resuelve el preset (los nombres desconocidos lanzan una excepción), añade un evento `permission/preset` de solo registro salvo que `name` ya sea el preset efectivo, y luego escribe cada knob a través de su propio setter — `setSandboxMode` de [dsh-sandbox-policy](../../packages/sandbox/sandbox-policy) y `setApprovalPolicy` de [dsh-user-approval](../../packages/interaction/user-approval) — solo cuando cambia el valor efectivo de ese knob. El evento de selección precede a los eventos de los knobs en el mismo turno, y volver a seleccionar el preset efectivo no añade nada en absoluto.

`permission/preset` es intención del usuario durable y de solo registro: se mantiene fuera del transcript del modelo (los eventos de los knobs son dueños de las consecuencias visibles para el modelo a través de sus consumidores), y existe para que `current()` pueda preservar QUÉ preset eligió el usuario cuando dos presets comparten un bundle; `effectivePermissionPreset(events)` pliega el último, y el replay no necesita estado de puesta al día. La declaración completa del evento está en el [catálogo de eventos del log de persistencia](../persistence-catalog.es.md); las firmas de los métodos están en el [catálogo de servicios](#ctxpermissionpresets--permissionpresetservice) generado.

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.es.md).

<a id="ctxpermissionpresets--permissionpresetservice"></a>

### `ctx.permissionPresets` — `PermissionPresetService`

Owns the deployment's permission presets and their write path. Requires a confining `ctx.shell` executor and `ctx.approval`; unmatched knob values are reported as CUSTOM_PRESET, not an error.

```ts cordis-catalog
/**
 * Resolve the preset matching the effective knob values. A still-matching
 * last selection wins shared-bundle ties; otherwise the first table match
 * wins, or {@link CUSTOM_PRESET} when no entry matches.
 * @param events - the session's events in log order.
 * @returns the effective preset name, or `custom` when nothing matches.
 */
current(events: readonly SessionEvent[]): string

/**
 * Build the whole select value for one folded knob state: every table
 * option in declaration order, `custom` appended exactly while derived.
 * @param state - the folded knob overrides.
 * @returns the `permissions` projection payload.
 */
selectFor(state: KnobState): PermissionSelect

/**
 * Resolve a preset's knob bundle.
 * @param name - the preset name to resolve.
 * @returns the configured bundle.
 * @throws when `name` is not in the table.
 */
resolve(name: string): PresetSpec

/**
 * Build the client option for a table entry or {@link CUSTOM_PRESET}. A
 * missing label falls back to the table key.
 * @param name - a table key, or `custom`.
 * @returns the option a client renders.
 * @throws when `name` is neither a table key nor `custom`.
 */
optionOf(name: string): PresetOption

/**
 * Record a changed preset, then update each changed knob through its own
 * setter. Selecting the effective preset again appends nothing.
 * @param session - the session the switch belongs to.
 * @param name - the preset to switch to; unknown names throw.
 */
set(session: Session, name: string): void
```

Types: [Session](session.es.md) · [SessionEvent](session.es.md)

Source: [`packages/interaction/permission-presets/src/index.ts`](../../packages/interaction/permission-presets/src/index.ts)
<!-- END GENERATED cordis-surface -->
