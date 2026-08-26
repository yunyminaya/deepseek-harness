<!-- Generado por scripts/gen-cordis-catalog.ts — no lo edites a mano.
     Ejecuta `pnpm run gen-cordis-catalog` para regenerarlo. -->

# Eventos

[English](events.md) | Español

La API de dispatch de eventos mezclada en cada contexto. Las declaraciones de eventos del harness y sus modos de dispatch se generan en cada [página de subsistema](../subsystems/core.es.md) propietaria.

### ctx.parallel(name, ...args)

```ts cordis-catalog
/**
 * Dispatch an event, running all listeners concurrently.
 *
 * @param name — the event name.
 * @param args — arguments passed to every listener.
 * @returns a promise resolving once every listener has settled.
 */
parallel<K extends keyof Events>(name: K, ...args: Parameters<Events[K]>): Promise<void>
parallel<K extends keyof Events>(thisArg: NoInfer<ThisType<Events[K]>>, name: K, ...args: Parameters<Events[K]>): Promise<void>
```

Despacha un evento, ejecutando todos los listeners de forma concurrente.

- `name` — el nombre del evento.
- `args` — argumentos pasados a cada listener.

**Devuelve** una promesa que se resuelve cuando todos los listeners han terminado.

[Source](../../vendor/cordis/src/events.ts#L44)

### ctx.emit(name, ...args)

```ts cordis-catalog
/**
 * Dispatch an event synchronously, ignoring listener return values.
 *
 * @param name — the event name.
 * @param args — arguments passed to every listener.
 */
emit<K extends keyof Events>(name: K, ...args: Parameters<Events[K]>): void
emit<K extends keyof Events>(thisArg: NoInfer<ThisType<Events[K]>>, name: K, ...args: Parameters<Events[K]>): void
```

Despacha un evento de forma síncrona, ignorando los valores de retorno de los listeners.

- `name` — el nombre del evento.
- `args` — argumentos pasados a cada listener.

[Source](../../vendor/cordis/src/events.ts#L53)

### ctx.serial(name, ...args)

```ts cordis-catalog
/**
 * Dispatch an event, awaiting listeners in order until one bails.
 *
 * @param name — the event name.
 * @param args — arguments passed to each listener.
 * @returns the first bail value (non-null, non-false, non-undefined), if any.
 */
serial<K extends keyof Events>(name: K, ...args: Parameters<Events[K]>): Promisify<ReturnType<Events[K]>>
serial<K extends keyof Events>(thisArg: NoInfer<ThisType<Events[K]>>, name: K, ...args: Parameters<Events[K]>): Promisify<ReturnType<Events[K]>>
```

Despacha un evento, esperando a los listeners en orden hasta que uno se retira.

- `name` — el nombre del evento.
- `args` — argumentos pasados a cada listener.

**Devuelve** el primer valor de retirada (bail) (no null, no false, no undefined), si lo hay.

[Source](../../vendor/cordis/src/events.ts#L63)

### ctx.bail(name, ...args)

```ts cordis-catalog
/**
 * Dispatch an event, calling listeners in order until one bails.
 *
 * @param name — the event name.
 * @param args — arguments passed to each listener.
 * @returns the first bail value (non-null, non-false, non-undefined), if any.
 */
bail<K extends keyof Events>(name: K, ...args: Parameters<Events[K]>): ReturnType<Events[K]>
bail<K extends keyof Events>(thisArg: NoInfer<ThisType<Events[K]>>, name: K, ...args: Parameters<Events[K]>): ReturnType<Events[K]>
```

Despacha un evento, llamando a los listeners en orden hasta que uno se retira.

- `name` — el nombre del evento.
- `args` — argumentos pasados a cada listener.

**Devuelve** el primer valor de retirada (bail) (no null, no false, no undefined), si lo hay.

[Source](../../vendor/cordis/src/events.ts#L73)

### ctx.waterfall(name, ...args)

```ts cordis-catalog
/**
 * Dispatch an event whose last argument is a `next` continuation.
 *
 * Each listener wraps the rest of the chain: calling `next()` invokes the
 * next listener (finally the built-in behavior); not calling it vetoes.
 *
 * @param name — the event name.
 * @param args — listener arguments; the final one is the innermost `next`.
 * @returns the outermost listener's return value.
 */
waterfall<K extends keyof Events>(name: K, ...args: Parameters<Events[K]>): ReturnType<Events[K]>
waterfall<K extends keyof Events>(thisArg: NoInfer<ThisType<Events[K]>>, name: K, ...args: Parameters<Events[K]>): ReturnType<Events[K]>
```

Despacha un evento cuyo último argumento es una continuación `next`.

Cada listener envuelve el resto de la cadena: llamar a `next()` invoca al siguiente listener (y finalmente el comportamiento integrado); no llamarlo veta.

- `name` — el nombre del evento.
- `args` — argumentos del listener; el último es el `next` más interno.

**Devuelve** el valor de retorno del listener más externo.

[Source](../../vendor/cordis/src/events.ts#L86)

### ctx.on(name, listener, options?)

```ts cordis-catalog
/**
 * Register an event listener owned by the current fiber.
 *
 * @param name — the event name to listen for.
 * @param listener — called with the dispatch arguments.
 * @param options — listener options; a boolean is shorthand for `prepend`.
 * @returns a disposer removing the listener; `true` if it was still registered.
 */
on<K extends keyof Events>(name: K, listener: Events[K], options?: boolean | EventOptions): () => boolean
```

Registra un listener de eventos propiedad del fiber actual.

- `name` — el nombre del evento que escuchar.
- `listener` — llamado con los argumentos del dispatch.
- `options` — opciones del listener; un booleano es una abreviatura de `prepend`.

**Devuelve** un disposer que elimina el listener; `true` si todavía estaba registrado.

[Source](../../vendor/cordis/src/events.ts#L97)

### ctx.once(name, listener, options?)

```ts cordis-catalog
/**
 * Same as `on()`, but the listener disposes itself after its first call.
 *
 * @param name — the event name to listen for.
 * @param listener — called at most once with the dispatch arguments.
 * @param options — listener options; a boolean is shorthand for `prepend`.
 * @returns a disposer removing the listener; `true` if it was still registered.
 */
once<K extends keyof Events>(name: K, listener: Events[K], options?: boolean | EventOptions): () => boolean
```

Igual que `on()`, pero el listener se autoelimina después de su primera llamada.

- `name` — el nombre del evento que escuchar.
- `listener` — llamado como mucho una vez con los argumentos del dispatch.
- `options` — opciones del listener; un booleano es una abreviatura de `prepend`.

**Devuelve** un disposer que elimina el listener; `true` si todavía estaba registrado.

[Source](../../vendor/cordis/src/events.ts#L106)

## EventOptions

Opciones aceptadas por `ctx.on()` y `ctx.once()`.

```ts cordis-catalog
/** Options accepted by `ctx.on()` and `ctx.once()`. */
interface EventOptions {
  /** Add the listener before existing listeners for the same event. */
  prepend?: boolean
  /** Receive the event regardless of context filter checks. */
  global?: boolean
}
```

[Source](../../vendor/cordis/src/events.ts#L112)

## DispatchMode

Estrategia de dispatch de eventos usada por el servicio de eventos.

`emit` ejecuta los listeners síncronos sin esperarlos, `parallel` espera a todos los listeners a la vez, `serial` los espera en orden hasta que uno se retira, `bail` se detiene en el primer valor de retirada síncrono y `waterfall` compone los listeners alrededor de un callback final `next`.

```ts cordis-catalog
/**
 * Event dispatch strategy used by the event service.
 *
 * `emit` runs synchronous listeners without awaiting them, `parallel` awaits
 * all listeners together, `serial` awaits them in order until one bails,
 * `bail` stops on the first synchronous bail value, and `waterfall` composes
 * listeners around a final `next` callback.
 */
type DispatchMode = 'emit' | 'parallel' | 'serial' | 'bail' | 'waterfall'
```

[Source](../../vendor/cordis/src/events.ts#L32)
