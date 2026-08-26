# Espacios de trabajo

[English](workspace.md) | Español

Un espacio de trabajo es el registro duradero de un directorio en el que trabaja el usuario: un id estable sobre una ruta canónica, un título de visualización y la relación ordenada de las sesiones que le pertenecen. El subsistema es un único paquete ([dsh-workspace](../../packages/workspace/workspace), `ctx.workspaceRegistry`) — una capacidad opcional del lado del host, no parte de la columna vertebral del agent loop (bucle del agente), e invisible para los modelos (sin herramientas, sin texto de prompt, sin eventos de sesión). Almacena sus registros a través de la [forma de dominio de almacenamiento](storage.es.md) y valida la pertenencia de las sesiones contra [`SessionHeader.cwd`](persistence.es.md#sessionheader--metadata-beside-the-log), de modo que `storageDomain` y `sessionPersistence` son dependencias de arranque obligatorias: un peer de persistencia no disponible deja el plugin en pending en lugar de que se lo confunda con un historial vacío. Registro de diseño: [Agent Note de almacenamiento KV por dominio](../../.agents/notes/proposed/architecture/2026-07-24-domain-kv-storage-and-workspace.es.md); arranque y orden de la GUI: [Agent Note de flujo de producto de Workspace UI](../../.agents/notes/implemented/feature/2026-07-25-workspace-ui-product-flow.es.md).

Fuente: [`packages/workspace/workspace/src/types.ts`](../../packages/workspace/workspace/src/types.ts)

## Identidad

```ts type-equiv
/**
 * Identifies one workspace record. A generated uuid, never the path: path
 * normalization rewrites paths, and a reference anchor must stay stable.
 */
type WorkspaceId = Branded<'WorkspaceId'>
```

`WorkspaceId` es un [id con marca](core.es.md#branded-ids). La identidad de ruta es aparte: `realpathNormalize` (`fs.realpath`; barras finales, `..` y enlaces simbólicos resueltos) es el único canon de unicidad — las rutas de los espacios de trabajo se almacenan canonicalizadas, la unicidad es la igualdad de cadenas de las rutas canónicas (un enlace simbólico a un directorio ya poseído colisiona), y las comprobaciones de cwd de sesión en el momento de vincular pasan por el mismo canon.

## La entidad de espacio de trabajo

Los consumidores solo ven la interfaz `Workspace`; la implementación permanece privada al paquete.

```ts type-equiv
/**
 * One workspace: a stable id over an existing directory, a display title, and
 * an ordered candidate account of sessions. Membership requires both an id in
 * that account and a session header whose canonical cwd equals the workspace
 * path. Consumers only see this interface; the implementation stays private.
 */
interface Workspace {
  /** Stable record id (generated uuid). */
  readonly id: WorkspaceId

  /**
   * Canonical directory path: the `fs.realpath` of the path given at create
   * time (trailing slashes, `..`, and symlinks all resolved). Never rewritten
   * afterwards, even when the directory disappears (see {@link status}).
   */
  readonly path: string

  /** Display title. Defaults to `basename(path)` at create; duplicates are allowed. */
  readonly title: string

  /** ISO-8601 creation instant, stamped at create and never rewritten. */
  readonly createdAt: string

  /** ISO-8601 instant of the last durable mutation (create counts as one). */
  readonly updatedAt: string

  /**
   * Header-validated sessions in manually owned order: a new session is
   * prepended at attach, explicit reordering goes through
   * `insertSessionBefore`, and activity never reorders. The durable candidate
   * account is filtered synchronously: missing headers, invalid cwd values,
   * and canonical cwd mismatches are never returned. A subsequent workspace
   * mutation prunes those filtered candidates durably.
   */
  readonly sessionIds: readonly SessionId[]

  /**
   * Replace the display title durably.
   * @param title - New title; any string, duplicates across workspaces allowed.
   * @returns resolution after durability.
   */
  setTitle(title: string): Promise<void>

  /**
   * Prepend a session to this workspace's candidate account. An already
   * accounted id resolves without writing, aside from the durable
   * filtered-candidate prune every accepted mutation performs. A new id's
   * live or persisted
   * header cwd must resolve to an existing directory equal to {@link path};
   * unknown ids, missing or invalid cwd values, and mismatches reject without
   * writing.
   * @param sessionId - The session to record.
   * @returns resolution after durability.
   */
  attachSession(sessionId: SessionId): Promise<void>

  /**
   * Move an accounted session within the manual order, DOM-insertBefore-like:
   * with an anchor the session lands before it, without one it appends to the
   * end. Only the moved id changes position. A session or anchor absent from
   * the account rejects without writing; a move to the current position
   * resolves without writing, aside from the durable filtered-candidate
   * prune every accepted mutation performs; decided on the domain write
   * chain.
   * @param sessionId - The accounted session to move.
   * @param beforeSessionId - Accounted anchor to insert before; omitted appends.
   * @returns resolution after durability.
   */
  insertSessionBefore(sessionId: SessionId, beforeSessionId?: SessionId): Promise<void>

  /**
   * Remove a session from this workspace's account. Idempotent: an id not on
   * the account resolves without writing, aside from the durable
   * filtered-candidate prune every accepted mutation performs; decided on
   * the domain write chain like attach. Never touches the session's own stored log.
   * @param sessionId - The session to remove.
   * @returns resolution after durability.
   */
  detachSession(sessionId: SessionId): Promise<void>

  /**
   * Live directory check, uncached: whether {@link path} currently exists and
   * is a directory. A missing directory never mutates the record — the
   * directory may only be temporarily moved.
   * @returns `'ok'` when the directory exists, `'missing-dir'` otherwise.
   */
  status(): Promise<'ok' | 'missing-dir'>
}
```

La verdad de la titularidad es la `sessionIds` ordenada del registro, nunca derivada del cwd de la sesión — pero la pertenencia exige ambas cosas: un id en la relación y un header cuyo cwd canónico iguale la ruta del espacio de trabajo, de modo que una sesión pertenece estructuralmente, como mucho, a un espacio de trabajo. Las escrituras fallidas rechazan (los errores de relación de `insertSessionBefore` como `WorkspaceMoveInvalidError`, los fallos de almacenamiento como errores planos); toda mutación aceptada sella `updatedAt` y poda de forma duradera los candidatos que ya no pasan la verificación de pertenencia.

## El registro: `ctx.workspaceRegistry`

`WorkspaceRegistry` ([firmas](#ctxworkspaceregistry--workspaceregistry)) es dueño de la inscripción y la resolución. `create(path, title?)` canonicaliza la ruta, rechaza una ruta inexistente (el `ENOENT` original) o un no-directorio, devuelve la entidad existente sin cambios cuando la ruta canónica ya está poseída y, en caso contrario, crea un registro con `title ?? basename(path)` antepuesto al orden duradero del registro — un registro nuevo no puede duplicar un título de visualización existente (`WorkspaceNameConflictError`). `get(id)` y el `list()` ordenado son lecturas síncronas de caché; `resolveByPath(path)` aplica el mismo canon de realpath sin crear. `delete(id)` elimina solo la inscripción, la entrada de orden y la relación de sesiones — el directorio, los archivos del usuario, las sesiones activas y los logs persistidos nunca se tocan, de modo que esas sesiones pasan a Ungrouped ([decisión](../../.agents/notes/implemented/feature/2026-07-27-workspace-registration-deletion.es.md)); los ids desconocidos devuelven `false`. Create y delete persisten un marcador de mutación pendiente antes de que sus dos escrituras (registro + orden) puedan divergir; el arranque resuelve exactamente la mutación marcada — eliminando la fila de tabla marcada, lo que completa un delete interrumpido y revierte un create interrumpido (la inscripción es recreable, así que la reversión es la dirección segura) — y un desajuste sin marcar entre orden y tabla falla de forma sonora como corrupción.

Las sesiones reciben su cwd en el momento de crearse, de quien las crea, no de este registro — el gateway de API resuelve el cwd de una sesión nueva a partir del `path` del espacio de trabajo elegido (con respaldo en un cwd explícito o por defecto), crea la sesión para que el cwd aterrice en su [`SessionHeader`](persistence.es.md#sessionheader--metadata-beside-the-log) inmutable, y luego llama a `attachSession`, que vuelve a validar ese cwd de header almacenado contra la ruta del espacio de trabajo. En el primer arranque con éxito, el registro hace el bootstrap del historial solo a partir de los headers persistidos (`id`, `cwd`, `createdAt` — nunca de los cuerpos de eventos), agrupando en espacios de trabajo por directorio las sesiones con un cwd canónico válido, las más nuevas primero; el marcador de inicializado se escribe al final para que un bootstrap interrumpido se reanude con seguridad. El bootstrap ocurre una sola vez: las sesiones heredadas sin cwd permanecen en Ungrouped, y las sesiones creadas después solo se unen a un espacio de trabajo mediante `attachSession`.

## Consumidores

[dsh-host-apiproxy](../../packages/host/apiproxy) es el consumidor de producto: sirve el CRUD de espacios de trabajo a los clientes GUI a través de `ctx.workspaceRegistry` y ejecuta el flujo de crear-sesión-luego-vincular descrito arriba. [dsh-agent-instructions](../../packages/context/agent-instructions) **no** es un consumidor a pesar del nombre: descubre archivos de instrucciones estilo AGENTS.md bajo el cwd del propio agent (agente) y nunca toca `ctx.workspaceRegistry` — la palabra compartida se refiere al directorio de trabajo del usuario, no a las entidades de este registro.

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxdirectorypicker--directorypicker-abstract-seam"></a>

### `ctx.directoryPicker` — `DirectoryPicker` (abstract seam)

Abstract directory-picking service. Subclass, implement `capability()`, and load the subclass as a plugin — it registers as `ctx.directoryPicker` (one implementation per context; loading a second throws, cordis' standard duplicate-service behavior). The capability object must be stable for the service lifetime: consumers may capture it across calls.

```ts cordis-catalog
/**
 * The backend's interaction capability.
 * @returns the discriminated capability consumers switch on.
 */
abstract capability(): DirectoryPickerCapability
```

Source: [`packages/host/directory-picker/src/index.ts`](../../packages/host/directory-picker/src/index.ts)

<a id="ctxworkspaceregistry--workspaceregistry"></a>

### `ctx.workspaceRegistry` — `WorkspaceRegistry`

Durable workspace registry. Startup waits for `sessionPersistence`, builds one canonical-cwd header index, and completes the one-time history bootstrap before the service becomes active. The persistence dependency is mandatory so an unavailable peer can never be mistaken for an empty history and commit the initialized marker.

```ts cordis-catalog
/**
 * Create or reuse a workspace for an existing directory. The path is
 * canonicalized through `fs.realpath`; a nonexistent path rejects with the
 * original error and a non-directory rejects. Repeated calls for the same
 * canonical path return the existing entity without changing its title.
 * A newly created workspace is prepended to the durable registry order.
 * Different canonical paths may share a display title.
 * @param path - Existing directory to own, in any path spelling.
 * @param title - Display title used only when a new record is created.
 * @returns the existing or newly durable workspace.
 */
async create(path: string, title?: string): Promise<Workspace>

/**
 * Look up a workspace by id.
 * @param id - Workspace id.
 * @returns the workspace, or `undefined` when unknown.
 */
get(id: WorkspaceId): Workspace | undefined

/**
 * Synchronous workspace projection in durable registry order. Every
 * entity's `sessionIds` getter is already filtered by the startup/live
 * canonical-cwd header index; this method performs no persistence reads.
 * @returns a fresh ordered array of workspace entities.
 */
list(): Workspace[]

/**
 * Delete one workspace registration while retaining its directory and every
 * session log. The durable order is updated before the table deletion; a
 * failed table write restores the prior order and keeps the entity
 * published. Unknown ids are an idempotent no-op for domain callers.
 * @param id - Workspace registration to remove.
 * @returns `true` when a record was deleted, `false` when it was unknown.
 */
delete(id: WorkspaceId): Promise<boolean>

/**
 * Move one workspace within the durable display order, DOM-insertBefore-like.
 * With an anchor it lands before that workspace; without one it appends.
 * @param id - Workspace to move.
 * @param beforeId - Workspace anchor; omitted appends.
 * @returns the complete committed workspace order.
 */
insertBefore(id: WorkspaceId, beforeId?: WorkspaceId): Promise<readonly WorkspaceId[]>

/**
 * Archive one session durably. The session must exist (live or in session
 * persistence); its workspace accounting — or lack of one — is irrelevant.
 * An already archived id resolves without writing.
 * @param sessionId - The session to archive.
 * @returns resolution after durability.
 */
archiveSession(sessionId: SessionId): Promise<void>

/**
 * Resolve by canonical directory path without creating or mutating a
 * workspace. A missing path rejects during `realpath`; an existing unowned
 * directory returns `undefined`.
 * @param path - Existing directory path in any spelling.
 * @returns the workspace owning the canonical path, when one exists.
 */
async resolveByPath(path: string): Promise<Workspace | undefined>
```

Types: [SessionId](core.es.md)

Source: [`packages/workspace/workspace/src/index.ts`](../../packages/workspace/workspace/src/index.ts)
<!-- END GENERATED cordis-surface -->
