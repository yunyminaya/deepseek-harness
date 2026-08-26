# Almacenamiento de spill

[English](spill.md) | Español

El seam de almacenamiento de spill — un [seam de capacidad](../../.agents/notes/implemented/architecture/2026-07-08-tool-output-spill-files.es.md) que persiste el texto sobredimensionado de una herramienta y devuelve un localizador orientado al modelo junto con una guía de recuperación — está repartido entre paquetes: la Service Definition ([dsh-spill](../../packages/spill/spill), `ctx.spillStore`), el Service Provider ([dsh-spill-local](../../packages/spill/spill-local), archivos privados con alcance de sesión en el sistema de archivos del host) y el Consumer ([dsh-spill-policy](../../packages/spill/spill-policy), la política `tools/post-execute`). Spill es **una capacidad opcional**, no parte del tronco del agent loop (bucle del agente) — por eso su vocabulario vive aquí, no en [core.md](core.es.md). Los mecanismos de vista previa siguen en [dsh-output-retention](../../packages/util/output-retention); este seam solo guarda el texto final que la política le entrega.

Fuente: [`packages/spill/spill/src/types.ts`](../../packages/spill/spill/src/types.ts)

## La solicitud de guardado

`saveText` es la única operación del servicio: persiste `content` tal cual, devuelve un localizador opaco, una pista de recuperación suministrada por el backend y el recuento exacto de bytes. La solicitud lleva el espacio de nombres de almacenamiento del momento del guardado (`owner`), la herramienta y la llamada que lo produjeron (`source`, usado para nombrar e inspeccionar — no para el control de acceso), y un `suggestedName` que el backend puede usar como pista de nombre (no es una ruta).

```ts type-equiv
/** One request to persist text to a spill artifact. */
interface SaveTextSpill {
  owner: SpillOwner
  source: SpillSource
  /**
   * A caller-suggested base name (e.g. `web_fetch.txt`). The backend sanitizes
   * it to a single safe path segment before use — it is a hint, never a path.
   */
  suggestedName: string
  /** The full text to persist (UTF-8). */
  content: string
}
```

```ts type-equiv
/**
 * Save-time storage namespace for a spilled artifact. The session id lets a
 * backend group storage under the producing session, but the returned
 * {@link SpillLocator} is the model-facing handle. Forked sessions inherit
 * locators already present in the seeded log; those artifacts are not copied or
 * re-owned, and spills produced after the fork use the child session id.
 */
interface SpillOwner {
  sessionId: SessionId
}
```

`SpillOwner.sessionId` es el espacio de nombres de almacenamiento del momento del guardado. Las sesiones bifurcadas heredan los localizadores de spill existentes del log sembrado; esos artefactos no se copian ni se re-asignan, y los spills producidos después de la bifurcación usan el id de la sesión hija. Una limpieza por período de retención puede caducar localizadores antiguos junto con otros artefactos antiguos de la sesión; el seam de spill no define una política de limpieza por sesión.

```ts type-equiv
/**
 * Tool and call that produced one spilled artifact — recorded by the backend for a readable
 * filename and inspection. Not interpreted for access control; purely
 * descriptive.
 */
interface SpillSource {
  /** The tool whose result was spilled (e.g. `web_fetch`). */
  toolName: string
  /** The model-issued call id the result belongs to. */
  callId: CallId
  /** A short human label for the artifact (e.g. `result`). */
  label: string
}
```

## El resultado

```ts type-equiv
/** A saved spill artifact: its locator, byte length, and backend-specific retrieval guidance. */
interface SpillRef {
  locator: SpillLocator
  bytes: number
  retrievalHint: string
}
```

`SpillLocator` es un identificador [branded](core.es.md#branded-ids) orientado al modelo que devuelve el backend. El backend local lo renderiza como una ruta del sistema de archivos; un backend remoto o de base de datos puede renderizar un URI, una clave o un token de comando. Los consumers lo tratan como opaco y lo renderizan con `retrievalHint` en lugar de asumir que `read` es siempre el mecanismo de recuperación correcto.

```ts type-equiv
/**
 * Opaque model-facing handle for one spilled artifact. A local backend may use a
 * filesystem path; a remote or database backend may use a URI or key. Consumers
 * render it with {@link SpillRef.retrievalHint}, but do not parse it.
 */
type SpillLocator = Branded<'SpillLocator'>
```

## El servicio

`SpillStore` (`ctx.spillStore`, definido en [`packages/spill/spill/src/index.ts`](../../packages/spill/spill/src/index.ts)) es un servicio abstracto de un solo método: `saveText(input) → Promise<SpillRef>`. Persiste el `content` COMPLETO y RECHAZA ante un fallo real de almacenamiento (permisos, ENOSPC, backend no disponible). El seam posee solo el almacenamiento: ni política de retención, ni reemplazo de resultados de herramienta, ni API de recuperación/búsqueda.

El backend local ([dsh-spill-local](../../packages/spill/spill-local)) escribe bajo `<root>/session-<hash>/<random>-<safeName>`: una raíz privada (0700) configurada o creada de forma perezosa, un subdirectorio de sesión `sha256(sessionId)` y una escritura exclusiva solo para el propietario (`open(path, 'wx', 0o600)`) para que un symlink plantado no pueda redirigirla. Su `locator` es la ruta local y su `retrievalHint` le dice al modelo que use `read` o `grep` sobre esa ruta. El consumer de política ([dsh-spill-policy](../../packages/spill/spill-policy)) reemplaza un resultado final de texto plano que supera `maxInlineBytes` con una vista previa de cabeza/cola de la librería de retención más la referencia del spill, en modo best-effort: un fallo de guardado conserva el resultado inline original en lugar de convertir una llamada exitosa en un `isError`.

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxspillstore--spillstore-abstract-seam"></a>

### `ctx.spillStore` — `SpillStore` (abstract seam)

Abstract spill storage service. Subclass, implement saveText, and load the subclass as a plugin — it registers as `ctx.spillStore` (one implementation per context; loading a second throws, cordis' standard duplicate-service behavior).

Semantics every implementation must honor:

- saveText persists the FULL `content` verbatim and returns an opaque locator, exact byte length, and model-facing retrieval guidance.
- Storage is scoped by the request's SaveTextSpill.owner session; the backend chooses a private (not world-readable) location and a collision-free name derived from — never equal to — the caller's `suggestedName`.
- `saveText` REJECTS on a real storage failure (permissions, ENOSPC, backend unavailable); the caller decides how to degrade (the spill policy treats a rejection as best-effort and keeps the inline result).

```ts cordis-catalog
/**
 * Persist `input.content` to a session-scoped spill artifact.
 * @param input - the owner, caller-supplied source fields, suggested name, and full text to save.
 * @returns the saved artifact's {@link SpillRef}; rejects on a storage failure.
 */
abstract saveText(input: SaveTextSpill): Promise<SpillRef>
```

Source: [`packages/spill/spill/src/index.ts`](../../packages/spill/spill/src/index.ts)
<!-- END GENERATED cordis-surface -->
