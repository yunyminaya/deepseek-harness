<!-- El archivo en inglés lo genera scripts/gen-cordis-catalog.ts; este archivo en español es la contraparte revisada, mantenida mediante el emparejamiento bilingüe.
     Para actualizarlo, ejecuta primero `pnpm run gen-cordis-catalog` para regenerar el inglés y, a continuación, actualiza este archivo y ejecuta `pnpm run verify-translation-pairing --write docs/cordis-api/fiber.md` para volver a registrar el par. -->

# Fiber

[English](fiber.md) | Español

Un fiber es una instancia de plugin cargada: su estado de ciclo de vida, su configuración validada y sus efectos registrados. `ctx.fiber` es el fiber actual y `ctx.effect()` delega en él.

### ctx.effect(execute, label?)

```ts cordis-catalog
/**
 * Register a cleanup-aware effect on this fiber.
 *
 * `execute` runs immediately; the disposers it produces are collected and
 * run (in reverse order) either when the returned disposer is called or
 * when the fiber unloads, whichever comes first. Calling the disposer twice
 * is a no-op. Throws `CordisError('INACTIVE_EFFECT')` if the fiber is
 * already disposed, and `TypeError` if `execute` returns an invalid shape.
 *
 * @param execute — the effect body; see {@link Effect} for accepted shapes.
 * @param label — effect label shown in `getEffects()` diagnostics.
 * @returns a disposer that tears the effect down and settles once done.
 */
effect(execute: () => SyncEffect, label?: string): Disposable<Promise<void>>
effect(execute: () => Effect, label?: string): AsyncDisposable<Promise<void>>
```

Registra en este fiber un efecto que gestiona su limpieza.

`execute` se ejecuta de inmediato; los disposers que produce se recogen y se ejecutan (en orden inverso) cuando se llama al disposer devuelto o cuando se descarga el fiber, lo que ocurra primero. Llamar al disposer dos veces no tiene ningún efecto. Lanza `CordisError('INACTIVE_EFFECT')` si el fiber ya está liberado y `TypeError` si `execute` devuelve una forma no válida.

- `execute` — el cuerpo del efecto; consulta `Effect` para ver las formas aceptadas.
- `label` — la etiqueta del efecto que se muestra en los diagnósticos de `getEffects()`.

**Devuelve** un disposer que desmonta el efecto y se resuelve una vez terminado.

[Fuente](../../vendor/cordis/src/fiber.ts#L415)

### ctx.fiber

```ts cordis-catalog
/** The fiber (plugin runtime instance) that owns this context. */
fiber: Fiber
```

El fiber (instancia de runtime del plugin) que posee este contexto.

[Fuente](../../vendor/cordis/src/fiber.ts#L12)

## La clase Fiber

Instancia de runtime de una aplicación de plugin.

Un fiber hace seguimiento del estado de dependencias, la configuración validada, los efectos de ciclo de vida y la limpieza del contexto de plugin devuelto por `ctx.plugin()`.

[Fuente](../../vendor/cordis/src/fiber.ts#L184)

### fiber.uid

```ts cordis-catalog
/** Unique id within the registry; 0 for the root fiber, `null` once disposed. */
public uid: number | null
```

Id único dentro del registro; 0 para el fiber raíz y `null` una vez liberado.

[Fuente](../../vendor/cordis/src/fiber.ts#L186)

### fiber.ctx

```ts cordis-catalog
/** The context this fiber's plugin runs in (extends the parent context). */
public readonly ctx: Context
```

El contexto en el que se ejecuta el plugin de este fiber (extiende el contexto padre).

[Fuente](../../vendor/cordis/src/fiber.ts#L188)

### fiber.config

```ts cordis-catalog
/** The validated plugin config (updated by `update()`). */
public config: any
```

La configuración validada del plugin (actualizada por `update()`).

[Fuente](../../vendor/cordis/src/fiber.ts#L190)

### fiber.state

```ts cordis-catalog
/** Current lifecycle state; transitions emit `internal/status`. */
public state
```

Estado de ciclo de vida actual; las transiciones emiten `internal/status`.

[Fuente](../../vendor/cordis/src/fiber.ts#L194)

### fiber.dispose

```ts cordis-catalog
/** Dispose this fiber: unload the plugin, then settle once cleanup finished. */
public readonly dispose: () => Promise<void>
```

Libera este fiber: descarga el plugin y, a continuación, se resuelve una vez terminada la limpieza.

[Fuente](../../vendor/cordis/src/fiber.ts#L196)

### fiber.store

```ts cordis-catalog
/** Snapshot of required service implementations while loaded; `undefined` otherwise. */
public store: Dict<Impl> | undefined
```

Instantánea de las implementaciones de servicio requeridas mientras esté cargado; en caso contrario, `undefined`.

[Fuente](../../vendor/cordis/src/fiber.ts#L198)

### fiber.inertia

```ts cordis-catalog
/** The in-flight load/unload transition, if one is currently running. */
public inertia: Promise<void> | undefined
```

La transición de carga/descarga en curso, si hay una ejecutándose.

[Fuente](../../vendor/cordis/src/fiber.ts#L200)

### fiber.name

```ts cordis-catalog
/** The plugin's display name, inherited from the nearest named ancestor, else `'root'`. */
get name()
```

El nombre visible del plugin, heredado del ancestro con nombre más cercano; si no, `'root'`.

[Fuente](../../vendor/cordis/src/fiber.ts#L336)

### fiber.assertActive()

```ts cordis-catalog
/**
 * Throw if the fiber has already been disposed.
 *
 * @returns nothing when the fiber is still active.
 * @throws {CordisError} `INACTIVE_EFFECT` when the fiber's uid has been cleared.
 */
assertActive()
```

Lanza una excepción si el fiber ya ha sido liberado.

**Devuelve** nada cuando el fiber sigue activo.

[Fuente](../../vendor/cordis/src/fiber.ts#L351)

### fiber.effect(execute, label?)

```ts cordis-catalog
/**
 * Register a cleanup-aware effect on this fiber.
 *
 * `execute` runs immediately; the disposers it produces are collected and
 * run (in reverse order) either when the returned disposer is called or
 * when the fiber unloads, whichever comes first. Calling the disposer twice
 * is a no-op. Throws `CordisError('INACTIVE_EFFECT')` if the fiber is
 * already disposed, and `TypeError` if `execute` returns an invalid shape.
 *
 * @param execute — the effect body; see {@link Effect} for accepted shapes.
 * @param label — effect label shown in `getEffects()` diagnostics.
 * @returns a disposer that tears the effect down and settles once done.
 */
effect(execute: () => SyncEffect, label?: string): Disposable<Promise<void>>
effect(execute: () => Effect, label?: string): AsyncDisposable<Promise<void>>
```

Registra en este fiber un efecto que gestiona su limpieza.

`execute` se ejecuta de inmediato; los disposers que produce se recogen y se ejecutan (en orden inverso) cuando se llama al disposer devuelto o cuando se descarga el fiber, lo que ocurra primero. Llamar al disposer dos veces no tiene ningún efecto. Lanza `CordisError('INACTIVE_EFFECT')` si el fiber ya está liberado y `TypeError` si `execute` devuelve una forma no válida.

- `execute` — el cuerpo del efecto; consulta `Effect` para ver las formas aceptadas.
- `label` — la etiqueta del efecto que se muestra en los diagnósticos de `getEffects()`.

**Devuelve** un disposer que desmonta el efecto y se resuelve una vez terminado.

[Fuente](../../vendor/cordis/src/fiber.ts#L415)

### fiber.getEffects()

```ts cordis-catalog
/**
 * Return metadata for currently registered effects.
 *
 * @returns one {@link EffectMeta} tree per labeled live effect.
 */
getEffects()
```

Devuelve los metadatos de los efectos registrados actualmente.

**Devuelve** un árbol `EffectMeta` por cada efecto vivo etiquetado.

[Fuente](../../vendor/cordis/src/fiber.ts#L568)

### fiber.await()

```ts cordis-catalog
/**
 * Wait for current lifecycle work and rethrow startup errors.
 *
 * @returns this fiber, once it has settled into a stable state.
 * @throws the config-validation or plugin-startup error, if any.
 */
async await()
```

Espera a que concluya el trabajo de ciclo de vida en curso y relanza los errores de arranque.

**Devuelve** este fiber, una vez que se ha asentado en un estado estable.

[Fuente](../../vendor/cordis/src/fiber.ts#L704)

### fiber.restart()

```ts cordis-catalog
/**
 * Dispose and immediately reload this plugin with its current config.
 *
 * @returns a promise resolving once the reload settled.
 * @throws {CordisError} `INACTIVE_EFFECT` when the fiber is already disposed.
 */
async restart()
```

Libera y recarga de inmediato este plugin con su configuración actual.

**Devuelve** una promesa que se resuelve cuando la recarga se asienta.

[Fuente](../../vendor/cordis/src/fiber.ts#L718)

### fiber.update(config, noSave?)

```ts cordis-catalog
/**
 * Validate and apply new config, then restart the plugin.
 *
 * Runs the `internal/update` waterfall first, so update hooks (and HMR)
 * can veto or replace the restart.
 *
 * @param config — the new raw config; validated before anything restarts.
 * @param noSave — hint for persistence hooks not to write the change back.
 * @returns the update waterfall result; the default restart returns a promise.
 * @throws when validation, an update listener, or the restarted plugin fails.
 */
update(config: any, noSave = false)
```

Valida y aplica la nueva configuración y, a continuación, reinicia el plugin.

Primero ejecuta el waterfall (cascada de eventos) de `internal/update`, de modo que los hooks de actualización (y el HMR, hot module replacement) puedan vetar o sustituir el reinicio.

- `config` — la nueva configuración en bruto; se valida antes de que se reinicie nada.
- `noSave` — indicación para que los hooks de persistencia no escriban el cambio.

**Devuelve** el resultado del waterfall de actualización; el reinicio por defecto devuelve una promesa.

[Fuente](../../vendor/cordis/src/fiber.ts#L736)

## Effect

Resultado del cuerpo de un efecto aceptado por `ctx.effect()` y por el arranque de plugins.

Puede ser un único disposer, una promesa de uno o un iterable (posiblemente asíncrono) que produce varios: los efectos generador registran cada disposer producido en cuanto se produce.

```ts cordis-catalog
/**
 * Effect body result accepted by `ctx.effect()` and plugin startup.
 *
 * Either a single disposer, a promise of one, or a (possibly async) iterable
 * yielding several — generator effects register each yielded disposer as it
 * is produced.
 */
type Effect<T = any> =
  | SyncEffect<T>
  | AsyncEffect<T>
```

[Fuente](../../vendor/cordis/src/fiber.ts#L83)

## Disposable

Función que un efecto devuelve para liberar recursos durante la liberación.

Los disposers se ejecutan en orden inverso al de registro cuando se descarga el fiber propietario; pueden ser asíncronos, en cuyo caso la descarga los espera.

```ts cordis-catalog
/**
 * Function returned by an effect to release resources during disposal.
 *
 * Disposers run in reverse registration order when the owning fiber unloads;
 * they may be async, in which case unloading awaits them.
 */
type Disposable<T = any> = () => T
```

[Fuente](../../vendor/cordis/src/fiber.ts#L74)

## EffectMeta

Nodo de árbol que se usa para exponer las etiquetas de efectos anidados en los diagnósticos.

```ts cordis-catalog
/** Tree node used to expose nested effect labels for diagnostics. */
interface EffectMeta {
  /** Human-readable effect label, e.g. `ctx.on("event")` or `ctx.provide("name")`. */
  label: string
  /** Metadata of nested effects registered while this effect ran. */
  children: EffectMeta[]
}
```

[Fuente](../../vendor/cordis/src/fiber.ts#L96)

## CordisError

Error del framework con un código estable legible por máquina.

```ts cordis-catalog
/** Framework error with a stable machine-readable code. */
class CordisError extends Error {
  /**
   * @param code — the stable error code; also the default message.
   * @param message — optional human-readable override.
   */
  constructor(public code: CordisError.Code, message?: string)
}

/** Cordis error code definitions. */
namespace CordisError {
  export type Code = keyof typeof Code

  export const Code = {
    INACTIVE_EFFECT: 'cannot create effect on inactive context',
  } as const
}
```

[Fuente](../../vendor/cordis/src/fiber.ts#L157)

## ValidationError

Error que se lanza cuando la configuración de un plugin no supera la validación de standard-schema.

```ts cordis-catalog
/** Error raised when plugin configuration fails standard-schema validation. */
class ValidationError extends TypeError {
  name = 'ValidationError'

  /**
   * Build the aggregated message from schema issues.
   *
   * @param issues — the standard-schema issues, one message line each.
   */
  constructor(issues: readonly StandardSchemaV1.Issue[])
}
```

[Fuente](../../vendor/cordis/src/fiber.ts#L19)
