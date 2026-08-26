# Sistema de archivos

[English](filesystem.md) | Español

La capacidad opcional de sistema de archivos tiene cuatro partes: [dsh-fs](../../packages/fs/fs) es dueño de `ctx.fs` y de las operaciones atómicas de texto con guardas opcionales, [dsh-fs-local](../../packages/fs/fs-local) implementa el disco local, [dsh-fs-observation-policy](../../packages/fs/fs-observation-policy) registra la presencia o ausencia observada y añade reglas de frescura mediante eventos en lugar de un servicio, y [dsh-tool-fs](../../packages/fs/tool-fs) ejecuta directamente las llamadas de lectura/escritura/edición orientadas al modelo y renderiza ventanas. Queda fuera del núcleo del agent loop (bucle del agente); los backends alternativos no cambian la política ni los schemas de las herramientas.

`dsh-fs-observation-policy` es opcional. Sin él, la Service Definition `FileSystem`, un provider y el Consumer `dsh-tool-fs` forman el seam de sistema de archivos completo y sin restricciones: `write` crea o sobrescribe incondicionalmente, y `edit` reemplaza incondicionalmente texto literal. El plugin de política cambia estas operaciones al decidir los waterfall (cascadas de eventos) de `fs/*`. Quitarlo no rompe la herramienta, porque la herramienta llama a `ctx.fs` y despacha eventos; no llama a los métodos de la política. Se espera que un despliegue que carga `dsh-tool-fs` cargue también `dsh-fs-observation-policy`, para que el comportamiento por defecto sea leer antes de escribir/editar.

Código fuente del provider: [`packages/fs/fs/src/types.ts`](../../packages/fs/fs/src/types.ts) y [`packages/fs/fs/src/index.ts`](../../packages/fs/fs/src/index.ts). Código fuente de la política: [`packages/fs/fs-observation-policy/src/types.ts`](../../packages/fs/fs-observation-policy/src/types.ts). Código fuente de la renderización de lectura: [`packages/fs/tool-fs/src/read-render.ts`](../../packages/fs/tool-fs/src/read-render.ts).

## Identidad del objetivo y metadatos (contrato del provider)

Toda operación resuelve primero una ruta suministrada por el usuario a un objetivo opaco del backend. Los Consumer pueden mostrar `displayPath`, pero no deben analizar `targetKey` (un id opaco con marca) ni suponer que es una ruta absoluta local.

Los Consumer que comparten el mundo de ejecución del sistema de archivos obtienen las coordenadas entre capacidades a través del provider en lugar de interpretar esa identidad: `processPath(target)` devuelve la ruta absoluta canónica que un subproceso puede abrir, `fileUrl(target)` devuelve su URI `file:` de la plataforma del provider, y `contains(parent, child)` comprueba la identidad canónica o la contención por descendencia.

```ts type-equiv
/**
 * A path resolved by a backend into a stable identity. `resolve()` produces
 * this; every other operation takes it.
 */
interface FsTarget {
  /** Opaque key for stale guards and target lookup. */
  targetKey: FsTargetKey
  /**
   * Path for model/UI-facing output. May be a local absolute path,
   * workspace-relative path, or remote URI depending on the backend.
   */
  displayPath: string
}
```

El backend es dueño de los tokens de versión de archivo — el token de frescura que una escritura/edición usa como guarda contra la desactualización. El plugin de política los guarda para las comprobaciones de desactualización; los Consumer no los interpretan. Ambos ids son cadenas opacas con marca.

```ts type-equiv
/**
 * Opaque key for stale guards and target lookup. The local backend uses a
 * realpath-like string; a remote backend might use a workspace URI or file id.
 * Consumers MUST NOT parse it or assume it is a local absolute path.
 */
type FsTargetKey = Branded<'FsTargetKey'>
```

```ts type-equiv
/**
 * Opaque file-version token — the freshness token a write/edit guards against.
 * The local backend derives it from high-resolution stat identity and freshness
 * fields; a remote backend might use a revision id. The policy layer records it
 * for stale checks; consumers may display related metadata but MUST NOT
 * interpret this token.
 */
type FsVersion = Branded<'FsVersion'>
```

`stat` devuelve metadatos (nunca contenido), o `undefined` cuando el objetivo no existe. `type` permite a los Consumer rechazar directorios y archivos especiales antes de leer, y `size` permite a los Consumer de texto elegir entre `readText` y `streamText` sin sondear por fallo. Un Consumer de texto aplica su propio tope de retención mientras consume `streamText`. Los Consumer de bytes crudos usan `readBytes(target, signal, maxBytes)`; su tope obligatorio de contenido completo hace que un desbordamiento conocido o descubierto falle con `FS_TOO_LARGE` en lugar de truncar o almacenar en búfer sin límite.

```ts type-equiv
/**
 * Metadata about a target — what {@link FileSystem.stat} returns. Lets the
 * policy layer reject directories/special files before reading and choose
 * `readText` vs `streamText` from `size` without probing by failure. `version`
 * is the freshness token. `undefined` from `stat` means the target is absent.
 */
interface FsInfo {
  /** Opaque freshness token of the target right now. */
  version: FsVersion
  /** Whether the target is a regular file, a directory, or something else. */
  type: 'file' | 'directory' | 'other'
  /** Byte size of a regular file, when the backend can report it. */
  size?: number
}
```

`lstat` es la primitiva de metadatos a nivel de ruta sin seguimiento. Toma una ruta en lugar de un `FsTarget` porque `resolve` sigue intencionadamente los enlaces simbólicos para producir una identidad estable; los Consumer que necesitan comprobaciones de límite de confianza pueden llamar a `lstat` primero y rechazar `symlink` antes de resolver.

```ts type-equiv
/**
 * Metadata about a path without following the final path component when it is a
 * symbolic link. Unlike {@link FsInfo}, this path-level probe can report
 * `symlink` so consumers with trust-boundary rules can reject repository-owned
 * links before resolving a target.
 */
interface FsPathInfo {
  /** Opaque freshness token of the path entry right now. */
  version: FsVersion
  /** Whether the path entry is a regular file, directory, symlink, or other. */
  type: 'file' | 'directory' | 'symlink' | 'other'
  /** Byte size of the path entry, when the backend can report it. */
  size?: number
}
```

`listDir` devuelve las entradas hijas directas en orden de nombre estable. Cada entrada lleva el nombre base del hijo, su tipo, el objetivo resuelto y metadatos baratos cuando el backend puede informarlos. No debe leer el contenido de los archivos, así que `size` es solo para archivos regulares y `version` se deriva de los metadatos. Los hijos rotos o desaparecidos pueden devolverse como `other` sin metadatos; los fallos de permisos o de E/S del backend al listar o resolver metadatos de hijos hacen fallar el listado completo con `FS_PERMISSION_DENIED` o `FS_IO_ERROR`.

```ts type-equiv
/**
 * One direct child returned by {@link FileSystem.listDir}. Listing returns
 * metadata and resolved targets only; it must not read file contents.
 */
interface FsDirEntry {
  /** Basename of the child inside the listed directory. */
  name: string
  /** Whether the child is a regular file, a directory, or something else. */
  type: 'file' | 'directory' | 'other'
  /** Resolved child target for follow-up operations. */
  target: FsTarget
  /** Opaque freshness token when the backend can report metadata cheaply. */
  version?: FsVersion
  /** Byte size of a regular file, when the backend can report it. */
  size?: number
}
```

## Guardas de escritura y edición (contrato del provider)

Tanto `writeText` como `editText` reciben su guarda de versión de forma OPCIONAL: omítela para una mutación incondicional (del provider básico), o provéela para proteger. La guarda de `writeText` es un `FsWriteIntent` — `createIfAbsent` crea un objetivo inexistente y rechaza uno existente con `FS_NOT_OBSERVED`, incluido un objetivo que aparece después del sondeo inicial del provider, porque la propia publicación debe ser sin reemplazo; `replaceIfVersion` reemplaza solo cuando el objetivo existe en la versión observada; si no, `FS_STALE_VERSION`. Omitir `expected` crea o sobrescribe incondicionalmente. La unión solo lleva las dos intenciones con guarda; «sin guarda» se expresa omitiendo, así que la escritura y la edición usan ambas el mismo campo `expected` opcional.

```ts type-equiv
/**
 * Guarded write intent. `createIfAbsent` rejects an existing target with
 * `FS_NOT_OBSERVED`; `replaceIfVersion` rejects absence or mismatch with
 * `FS_STALE_VERSION`. Omitting the intent from `writeText` means unconditional
 * create-or-overwrite, not a third union arm.
 */
type FsWriteIntent =
  | { kind: 'createIfAbsent' }
  | { kind: 'replaceIfVersion'; version: FsVersion }
```

```ts type-equiv
/** Outcome of a full-file write. */
interface FsWriteOutcome {
  /** Whether the write created a new file or replaced an existing one. */
  operation: 'create' | 'update'
  /** Opaque version of the file after the write. */
  version: FsVersion
  /**
   * The file's content BEFORE the write, or `null` when the file did not exist
   * (a create) or the backend declined a contextual basis (for example, a
   * binary/non-UTF-8 prior file or either overwrite side reaching its exclusive limit).
   * LF-normalized storage text (the diff basis), never a diff — a consumer
   * computes the result-time contextual diff from `before`/`after` when
   * `before` is present, else falls back to a whole-file diff.
   */
  before: string | null
  /** The file's content AFTER the write, LF-normalized to share `before`'s diff basis. */
  after: string
}
```

`editText` es una mutación a nivel de provider, no un `read` más un `write` compuestos en otro lugar. Con guarda, verifica la versión esperada ANTES de la coincidencia literal (así una edición desactualizada informa `FS_STALE_VERSION`, no un fallo de coincidencia contra contenido más nuevo); sin guarda, edita el contenido actual. En ambos casos aplica el reemplazo y escribe atómicamente — manteniendo la coincidencia, el manejo de finales de línea, la comprobación de desactualización y el reemplazo atómico dentro de una única sección crítica de la mutación — y un objetivo inexistente informa `FS_STALE_VERSION` en ambos caminos.

```ts type-equiv
/** A literal-replacement edit request. */
interface FsEditRequest {
  /** Literal non-empty text to replace. Must match exactly (after line-ending normalization). */
  oldString: string
  /** Literal replacement text. An empty string deletes the matched text. */
  newString: string
  /** Replace every match instead of requiring exactly one. */
  replaceAll: boolean
}
```

```ts type-equiv
/** Outcome of a literal edit. */
interface FsEditOutcome {
  /** Opaque version of the file after the edit. */
  version: FsVersion
  /**
   * The file's content BEFORE the edit. Raw storage text (LF-normalized by the
   * backend), never a diff — a consumer computes the result-time contextual diff
   * (the applied hunk with context) from `before`/`after`.
   */
  before: string
  /** The file's content AFTER the edit. */
  after: string
}
```

## Los eventos de política de fs (vocabulario del contrato del provider)

`dsh-fs` es dueño de tres eventos que la herramienta despacha y que el plugin de política escucha, de modo que el emisor (`dsh-tool-fs`) y el listener (`dsh-fs-observation-policy`) comparten un vocabulario sin que el emisor dependa del plugin de política. Solo llevan vocabulario de `dsh-fs` más un actor `object` opaco — sin conceptos orientados al modelo y sin estructura de dueño agent/sesión.

`fs/write-intent` y `fs/edit-intent` son **waterfall (cascadas de eventos) de decisión de un solo slot**: la herramienta despacha cada uno con un thunk por defecto que devuelve `undefined` (el provider básico), y un listener decide por completo sin llamar a `next()`. El slot es de primero que gana por orden de registro — que el plugin de política sea su dueño es una convención de despliegue, no un invariante impuesto. `fs/observed` es un evento de registro de disparar y olvidar que lleva un `FsObservation`: presente en una versión o ausencia confirmada. Se despacha con un `ctx.emit` simple; su listener DEBE ser síncrono y solo con efectos secundarios, porque la herramienta NO protege la emisión — un listener que lanza una excepción puede reemplazar un error de lectura o aparecer como resultado `isError` de la herramienta después de que una mutación ya haya tenido éxito. La [superficie cordis](#cordis-surface) generada más abajo muestra las firmas exactas.

```ts type-equiv
/**
 * One authoritative observation of a target. A present observation carries the
 * version used by guarded replacement; an absent observation authorizes only a
 * guarded create, never an edit.
 */
type FsObservation =
  | { readonly kind: 'present'; readonly version: FsVersion }
  | { readonly kind: 'absent' }
```

## Contexto de ejecución (plugin de política)

El plugin de política necesita solo el contexto de ejecución justo para derivar el dueño del estado observado, estrechando el actor `object` opaco que llevan los eventos `fs/*`. `ToolExecution` tiene los campos necesarios, así que `dsh-tool-fs` pasa su objeto de ejecución como actor sin obligar a `dsh-fs-observation-policy` a importar los paquetes de herramienta, agent o sesión.

```ts type-equiv
/**
 * Minimal structural view of a tool execution the policy plugin needs to derive
 * an observed-state owner. `@deepseek-ai/dsh-tools`' `ToolExecution` contains
 * these fields, so the tool passes its `exec` straight through as the opaque
 * `object` actor on the `fs/*` events; this plugin narrows that actor to
 * `FsObservationActor` without importing `dsh-tools`, `dsh-agent`, or `dsh-session`.
 *
 * The owner is `agent.session` when present. It is treated as an opaque object
 * identity (a `WeakMap` key); this package never reads any of its fields.
 */
interface FsObservationActor {
  /** The agent on whose behalf the call runs, when there is one. */
  agent?: {
    /** The session that owns observed-file state, used as an opaque key. */
    session?: object
  }
}
```

## Resultado de la lectura (Consumer / renderización de la lectura)

Una lectura de texto está acotada por la ventana de líneas, el tope de bytes y los límites del backend. Al alcanzar el tope de bytes, el escaneo continúa sin retener más líneas para que `totalLines` siga siendo exacto. El resultado que renderiza la herramienta `read` orientada al modelo es puramente presentacional; no existe una vista `full`/`partial` — la autorización se basa en la frescura (la herramienta emite un `fs/observed` presente directamente con la versión del stat), así que cualquier lectura con ventana puede autorizar una escritura/edición posterior cuando el archivo no ha cambiado. Un fallo de metadatos emite una observación de ausencia antes de que la herramienta devuelva `FS_NOT_FOUND`, lo que permite que una escritura con guarda posterior recree un objetivo eliminado externamente sin autorizar la edición. `dsh-tool-fs`, el ejecutor dueño de la lectura, implementa el ventaneo de lectura y construye este resultado; el plugin de política no.

```ts type-equiv
/** Outcome of a bounded text read — what {@link formatReadOutput} renders. */
interface FileReadOutcome {
  /** 1-based first line requested. */
  offset: number
  /** Returned lines, already numbered. */
  lines: FileTextLine[]
  /** Exact total line count in the file. */
  totalLines: number
  /** Whether selected output hit the byte cap. */
  truncatedByBytes?: true
}
```

## Estado de archivo observado (plugin de política)

El estado observado es un `WeakMap<owner, Map<targetKey, FsObservation>>` guardado dentro del plugin `dsh-fs-observation-policy`. Una entrada de mapa ausente significa no visto; `{ kind: 'absent' }` significa que un fallo de metadatos de un `read` o de `view`, `str_replace` o `insert` de `str_replace_editor` confirmó la ausencia; `{ kind: 'present', version }` significa que una lectura, escritura o edición observó esa versión. La decisión de escritura mapea no visto y ausente a `createIfAbsent`, y presente a `replaceIfVersion`; la decisión de edición mapea no visto a `FS_NOT_OBSERVED`, ausente a `FS_NOT_FOUND`, y presente a su guarda de versión. El dueño se deriva del actor del evento (normalmente `exec.agent.session`), se trata como opaco y nunca se lee. La disposición lo descarta todo (seguridad HMR), y la política no realiza ninguna E/S de sistema de archivos.

## Taxonomía de errores (contrato del provider)

Los fallos del sistema de archivos usan cadenas `FsErrorCode` estables transportadas por `FsError` (`HarnessError`). El registro de herramientas conserva `{ name, code }` en los resultados de error, de modo que las capas de reintento, permiso e interfaz pueden ramificar sin analizar texto.

```ts type-equiv
/**
 * Stable, machine-routable codes for filesystem failures. Carried on
 * {@link FsError}; the tool registry exposes `{ name, code }` on `isError`
 * results so retry/permission/UI layers can branch without parsing messages.
 */
type FsErrorCode =
  | 'FS_NOT_FOUND'
  | 'FS_NOT_DIRECTORY'
  | 'FS_NOT_TEXT'
  | 'FS_NOT_REGULAR_FILE'
  | 'FS_TOO_LARGE'
  | 'FS_PERMISSION_DENIED'
  | 'FS_SANDBOX_DENIED'
  | 'FS_IO_ERROR'
  | 'FS_STALE_VERSION'
  | 'FS_NOT_OBSERVED'
  | 'FS_AMBIGUOUS_EDIT'
  | 'FS_EDIT_NOT_FOUND'
  | 'FS_ABORTED'
```

`FS_NOT_DIRECTORY`, `FS_PERMISSION_DENIED` y `FS_IO_ERROR` los usa el listado de directorios para distinguir un objetivo existente que no es un directorio, un listado denegado y un fallo inesperado de E/S del backend. `FS_SANDBOX_DENIED` es una negativa de POLÍTICA de un backend que aplica sandbox (`dsh-fs-sandbox`) — la valla de modo denegó una escritura/edición — distinta de `FS_PERMISSION_DENIED` (el kernel del host que lo niega). `FS_NOT_OBSERVED` significa que el plugin de política no tiene registro de observación previa para este dueño (o que un `createIfAbsent` chocó con un archivo existente). `FS_NOT_FOUND` representa también una edición rechazada por ausencia confirmada. `FS_STALE_VERSION` significa que la versión del backend ya no coincide con la observada (o que el propio provider recibe una edición para un objetivo inexistente). La autorización por frescura no distingue parcial/completo, así que no existe `FS_PARTIAL_OBSERVATION`.

## Sin timeouts en la E/S de archivos

`read`/`write`/`edit` **no** reciben `timeoutMs`, y el contrato del provider no establece ningún plazo — a diferencia de bash y web (que consumen [`@deepseek-ai/dsh-timeout`](../../packages/util/timeout/README.es.md)) y de `glob`/`grep` respaldados por subprocesos (cuyo `timeoutMs` declarado lo aplica `@deepseek-ai/dsh-tool-call-timeout-policy`): esos están respaldados por procesos, donde un plazo puede de verdad matar el trabajo. Una llamada al sistema local es como mucho abortable en modo de mejor esfuerzo — un timeout no podría forzar a un `fsync`/`rename` en curso a detenerse, así que un `timeoutMs` aquí sería un plazo que el seam no puede aplicar, y un valor por defecto implícito en el lugar exacto donde lo explícito-sobre-lo-implícito lo prohíbe. La cancelación sigue propagándose a través de la signal de ejecución de la herramienta para un aborto de mejor esfuerzo en los límites de las llamadas al sistema.

## El servicio y el plugin

`FileSystem` (`ctx.fs`, abstracto) es dueño de las primitivas del provider: `resolve`, `processPath`, `fileUrl`, `contains`, `stat`, `lstat`, `readText`, `streamText`, `readBytes`, `listDir`, `writeText` y `editText`. `dsh-fs-observation-policy` no registra **ningún servicio** — es un plugin que añade política a través de la puerta de eventos `fs/*`: decide los waterfall de intención de escritura/edición a partir del estado no visto/ausente/presente y registra valores `FsObservation`. El ejecutor es `dsh-tool-fs`: lee/escribe/edita a través de `ctx.fs`, despacha los waterfall y emite el evento de registro. La [sección `ctx.fs`](#ctxfs--filesystem-abstract-seam) generada más abajo muestra las firmas exactas.

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxfs--filesystem-abstract-seam"></a>

### `ctx.fs` — `FileSystem` (abstract seam)

Abstract filesystem provider. Targets must preserve identity across aliases; reads expose regular UTF-8 text or typed errors, listings are stable and content-free, and mutations are atomic. Optional guards add stale protection without changing the unguarded provider contract.

```ts cordis-catalog
/**
 * Resolve a model/plugin-supplied path into a stable {@link FsTarget}. May perform I/O (a
 * remote/sandboxed backend may need a round-trip to map a path to a stable identity), hence
 * async even though the local backend only normalizes + realpaths.
 *
 * @param path - the path to resolve; relative paths resolve against `opts.cwd`.
 * @param opts - optional cwd override and cancellation signal.
 * @returns the stable target; the same file yields the same `targetKey`.
 */
abstract resolve(path: string, opts?: { cwd?: string; signal?: AbortSignal }): Promise<FsTarget>

/**
 * Return the canonical absolute path a subprocess in this filesystem's
 * execution world can open. The path is deliberately separate from
 * {@link FsTarget.targetKey}: consumers may pass this value to another OS
 * capability, but must continue treating the target key as opaque.
 * @param target - the resolved target whose process path is required.
 * @returns an absolute path in the backend's execution world.
 */
abstract processPath(target: FsTarget): string

/**
 * Return the canonical `file:` URI for a target in this filesystem's
 * execution world. Backends own URI encoding because the host platform may
 * differ from the execution platform.
 * @param target - the resolved target to encode.
 * @returns the target's canonical file URI.
 */
abstract fileUrl(target: FsTarget): string

/**
 * Test canonical containment without exposing or parsing backend target
 * keys. Both targets must come from this provider.
 * @param parent - canonical directory target.
 * @param child - canonical candidate target.
 * @returns true when `child` is `parent` or a descendant of it.
 */
abstract contains(parent: FsTarget, child: FsTarget): boolean

/**
 * Return target metadata, or `undefined` when the target does not exist.
 * @param target - the resolved target to stat.
 * @param signal - aborts the metadata round-trip.
 * @returns metadata only, never content; undefined for an absent target.
 */
abstract stat(target: FsTarget, signal?: AbortSignal): Promise<FsInfo | undefined>

/**
 * Return path metadata without following the final path component when it is a
 * symbolic link. This is intentionally path-shaped, not target-shaped:
 * {@link resolve} follows symlinks to produce the stable identity used by
 * normal reads/writes, while `lstat` lets a consumer reject the path itself
 * before that follow happens.
 *
 * `opts.cwd` follows {@link resolve}'s cwd rules. `undefined` means the path is
 * absent.
 * @param path - the path to inspect; relative paths resolve against `opts.cwd`.
 * @param opts - `cwd` overrides the backend's default base for relative paths.
 * @param signal - aborts the metadata round-trip.
 * @returns metadata only, never content; undefined for an absent path.
 */
abstract lstat(path: string, opts?: { cwd?: string }, signal?: AbortSignal): Promise<FsPathInfo | undefined>

/**
 * Read the whole regular text file as a single decoded string.
 * @param target - the resolved target to read.
 * @param signal - aborts the read.
 * @returns the full decoded UTF-8 content.
 */
abstract readText(target: FsTarget, signal?: AbortSignal): Promise<string>

/**
 * Stream the whole regular text file as decoded text chunks (same text
 * semantics as {@link readText}, for large files). The backend owns
 * cross-chunk UTF-8 decoding and binary rejection so the policy layer never
 * touches raw bytes.
 * @param target - the resolved target to read.
 * @param signal - aborts the stream, including between chunks.
 * @returns the chunk iterable, decoded and validated like {@link readText}.
 */
abstract streamText(target: FsTarget, signal?: AbortSignal): Promise<AsyncIterable<string>>

/**
 * Read the whole regular file as raw bytes with no decoding or binary
 * rejection. The bound lives at this seam so a backend can never buffer an
 * unbounded file: a target known or discovered to exceed `maxBytes` fails
 * with `FS_TOO_LARGE` instead of returning a truncated result.
 * @param target - the resolved target to read.
 * @param signal - aborts the read.
 * @param maxBytes - inclusive byte cap on the complete content.
 * @returns the full raw content, at most `maxBytes` long.
 */
abstract readBytes(target: FsTarget, signal: AbortSignal | undefined, maxBytes: number): Promise<Uint8Array>

/**
 * List direct children of a directory in stable name order. Returns resolved
 * child targets plus cheap metadata only; never reads file contents.
 * @param target - the resolved directory target.
 * @param signal - aborts the listing.
 * @returns one entry per direct child, in stable name order.
 */
abstract listDir(target: FsTarget, signal?: AbortSignal): Promise<FsDirEntry[]>

/**
 * Atomically create or replace UTF-8 text. `expected` guards intent and
 * staleness; omission allows unconditional overwrite.
 * @param target - the resolved target to write.
 * @param content - the full new file content.
 * @param expected - the write intent guarding the write; omit for unconditional.
 * @param signal - aborts before atomic publication takes effect.
 * @param sandboxPolicy - the per-call mode and workspace root this write
 *   runs under; a sandboxing backend fences the write by it, the bare backend
 *   ignores it. Omit to leave the backend its own default.
 * @returns the outcome, including the version the write produced.
 */
abstract writeText( target: FsTarget, content: string, expected?: FsWriteIntent, signal?: AbortSignal, sandboxPolicy?: SandboxExecutionPolicy, ): Promise<FsWriteOutcome>

/**
 * Atomically edit literal text. When supplied, the version guard is checked
 * before matching so stale content reports `FS_STALE_VERSION`; omission edits
 * the current content without a freshness precondition.
 * @param target - the resolved target to edit.
 * @param edit - the literal search/replace request.
 * @param expected - the version guard; omit for an unconditional edit.
 * @param signal - aborts before atomic publication takes effect.
 * @param sandboxPolicy - the per-call mode and workspace root this edit runs
 *   under; a sandboxing backend fences the edit by it, the bare backend
 *   ignores it. Omit to leave the backend its own default.
 * @returns the outcome, including the version the edit produced.
 */
abstract editText( target: FsTarget, edit: FsEditRequest, expected?: { version: FsVersion }, signal?: AbortSignal, sandboxPolicy?: SandboxExecutionPolicy, ): Promise<FsEditOutcome>
```

Types: [SandboxExecutionPolicy](sandbox.es.md)

Source: [`packages/fs/fs/src/index.ts`](../../packages/fs/fs/src/index.ts)

<a id="fs-events"></a>

### `fs/*` events

<a id="fsedit-intent--waterfall"></a>

#### `fs/edit-intent` — waterfall

Single-slot decision for the next FileSystem.editText. Calling `next()` yields an unconditional edit; the first returned guard wins.

```ts cordis-catalog
/**
 * Single-slot decision for the next {@link FileSystem.editText}. Calling
 * `next()` yields an unconditional edit; the first returned guard wins.
 * @param target - the resolved target about to be edited.
 * @param actor - the opaque tool-execution context the decider keys off.
 * @mode waterfall
 */
'fs/edit-intent'(target: FsTarget, actor: object | undefined, next: () => { version: FsVersion } | undefined | Promise<{ version: FsVersion } | undefined>): Promise<{ version: FsVersion } | undefined>
```

Source: [`packages/fs/fs/src/index.ts`](../../packages/fs/fs/src/index.ts)

<a id="fsobserved--emit"></a>

#### `fs/observed` — emit

Record an authoritative positive or negative observation. Listeners must be synchronous recorders: throws fail the tool call and returned promises are not awaited.

```ts cordis-catalog
/**
 * Record an authoritative positive or negative observation. Listeners must
 * be synchronous recorders: throws fail the tool call and returned promises
 * are not awaited.
 * @param target - the target whose presence or absence was observed.
 * @param observation - present with its version, or confirmed absent.
 * @param actor - the observing tool-execution context; undefined records nothing useful.
 * @mode emit
 */
'fs/observed'(target: FsTarget, observation: FsObservation, actor: object | undefined): void
```

Source: [`packages/fs/fs/src/index.ts`](../../packages/fs/fs/src/index.ts)

<a id="fswrite-intent--waterfall"></a>

#### `fs/write-intent` — waterfall

Single-slot decision for the next FileSystem.writeText. Calling `next()` yields the bare provider's unconditional write; the first listener that returns an intent owns the decision rather than composing with peers.

```ts cordis-catalog
/**
 * Single-slot decision for the next {@link FileSystem.writeText}. Calling
 * `next()` yields the bare provider's unconditional write; the first listener
 * that returns an intent owns the decision rather than composing with peers.
 * @param target - the resolved target about to be written.
 * @param actor - the opaque tool-execution context the decider keys off.
 * @mode waterfall
 */
'fs/write-intent'(target: FsTarget, actor: object | undefined, next: () => FsWriteIntent | undefined | Promise<FsWriteIntent | undefined>): Promise<FsWriteIntent | undefined>
```

Source: [`packages/fs/fs/src/index.ts`](../../packages/fs/fs/src/index.ts)
<!-- END GENERATED cordis-surface -->
