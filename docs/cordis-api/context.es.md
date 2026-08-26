<!-- Generado por scripts/gen-cordis-catalog.ts — no lo edites a mano.
     Ejecuta `pnpm run gen-cordis-catalog` para regenerarlo. -->

# Contexto

[English](context.md) | Español

El contexto es el objeto central de Cordis: todos los servicios, eventos y APIs del ciclo de vida se alcanzan a través de `ctx`. Los métodos de eventos se documentan en [Eventos](events.es.md), los efectos y el fiber actual en [Fiber](fiber.es.md) y la carga de plugins en [Registro](registry.es.md).

Contenedores de dependencias raíz e hijos para los plugins de Cordis.

Un contexto es un proxy: las lecturas normales de propiedades pasan por el resolver de servicios, mientras que `extend()`, `isolate()` e `intercept()` crean contextos hijos con ámbito sin mutar a su padre.

[Source](../../vendor/cordis/src/context.ts#L42)

### ctx.extend(meta?)

```ts cordis-catalog
/**
 * Create a child context with extra metadata on top of the current scope.
 *
 * The child prototypally inherits every property of this context; own
 * properties of `meta` shadow the inherited ones. The parent is not mutated.
 *
 * @param meta — own properties (including symbol keys) to define on the child.
 * @returns a child context inheriting from this one.
 */
extend(meta = {}): this
```

Crea un contexto hijo con metadatos extra sobre el ámbito actual.

El hijo hereda prototípicamente todas las propiedades de este contexto; las propiedades propias de `meta` sombrean las heredadas. El padre no se muta.

- `meta` — propiedades propias (incluidas claves de símbolo) que definir en el hijo.

**Devuelve** un contexto hijo que hereda de este.

[Source](../../vendor/cordis/src/context.ts#L99)

### ctx.isolate(name, label?)

```ts cordis-catalog
/**
 * Create a child context with an independent service scope for `name`.
 *
 * Below the returned context, reads and writes of the service `name`
 * resolve against the new label instead of the parent's, so a different
 * implementation can be provided without affecting the parent scope.
 * Passing the same `label` to two `isolate()` calls joins their scopes.
 *
 * @param name — the service name to isolate.
 * @param label — scope label to join; defaults to a fresh unique symbol.
 * @returns a child context whose `name` service resolves in the new scope.
 */
isolate(name: string, label?: symbol)
```

Crea un contexto hijo con un ámbito de servicio independiente para `name`.

Por debajo del contexto devuelto, las lecturas y escrituras del servicio `name` se resuelven contra la nueva etiqueta en lugar de la del padre, de modo que se puede aportar una implementación distinta sin afectar al ámbito del padre. Pasar la misma `label` a dos llamadas de `isolate()` une sus ámbitos.

- `name` — el nombre del servicio que aislar.
- `label` — etiqueta de ámbito a la que unirse; por defecto, un símbolo único nuevo.

**Devuelve** un contexto hijo cuyo servicio `name` se resuelve en el nuevo ámbito.

[Source](../../vendor/cordis/src/context.ts#L121)

### ctx.intercept(name, config)

```ts cordis-catalog
/**
 * Add service-specific intercept config for plugins started below this
 * context.
 *
 * Plugins loaded under the returned context see `config` merged into the
 * service's resolved config (ancestor entries first; see
 * `Service[symbols.resolveConfig]`). The parent context is not affected.
 *
 * @param name — the service name whose config to intercept.
 * @param config — the intercept config to merge for that service.
 * @returns a child context carrying the additional intercept entry.
 */
intercept<K extends InjectKey>(name: K, config: Context[K] extends { [symbols.config]: infer T } ? T : never): this
intercept(name: string, config: any): this
```

Añade configuración de interceptación específica de servicio para los plugins iniciados por debajo de este contexto.

Los plugins cargados bajo el contexto devuelto ven `config` fusionada en la configuración resuelta del servicio (las entradas de los ancestros primero; consulta `Service[symbols.resolveConfig]`). El contexto padre no se ve afectado.

- `name` — el nombre del servicio cuya configuración interceptar.
- `config` — la configuración de interceptación que fusionar para ese servicio.

**Devuelve** un contexto hijo que lleva la entrada de interceptación adicional.

[Source](../../vendor/cordis/src/context.ts#L139)

### ctx.root

```ts cordis-catalog
/** The root context of the application (every child context shares it). @experimental */
root: this
```

El contexto raíz de la aplicación (todo contexto hijo lo comparte). @experimental

[Source](../../vendor/cordis/src/context.ts#L22)

### ctx.baseUrl

```ts cordis-catalog
/** Base URL used to resolve relative plugin/module specifiers, if the runtime sets one. */
baseUrl?: string
```

URL base usada para resolver los especificadores relativos de plugins/módulos, si el runtime establece una.

[Source](../../vendor/cordis/src/context.ts#L24)

### ctx.events

```ts cordis-catalog
/** The event bus. Its methods are also mixed onto `ctx` (`ctx.on`, `ctx.emit`, ...). */
events: EventsService
```

El bus de eventos. Sus métodos también se mezclan (mixin) en `ctx` (`ctx.on`, `ctx.emit`, ...).

[Source](../../vendor/cordis/src/context.ts#L26)

### ctx.logger

```ts cordis-catalog
/** The logging service. Call `ctx.logger(name)` for a named logger. */
logger: LoggerService
```

El servicio de registro. Llama a `ctx.logger(name)` para un logger con nombre.

[Source](../../vendor/cordis/src/context.ts#L28)

### ctx.reflect

```ts cordis-catalog
/** The reflection layer backing the context proxy (`ctx.get`, `ctx.provide`, ...). */
reflect: ReflectService
```

La capa de reflexión que respalda el proxy del contexto (`ctx.get`, `ctx.provide`, ...).

[Source](../../vendor/cordis/src/context.ts#L30)

### ctx.registry

```ts cordis-catalog
/** The plugin registry. Its methods are mixed onto `ctx` (`ctx.plugin`, `ctx.inject`). */
registry: RegistryService
```

El registro de plugins. Sus métodos se mezclan (mixin) en `ctx` (`ctx.plugin`, `ctx.inject`).

[Source](../../vendor/cordis/src/context.ts#L32)

## Miembros estáticos

### Context.effect

```ts cordis-catalog
/** Symbol key under which a disposer exposes its {@link EffectMeta} diagnostics tree. */
static readonly effect: unique symbol
```

Clave de símbolo bajo la que un disposer expone su árbol de diagnósticos EffectMeta.

[Source](../../vendor/cordis/src/context.ts#L44)

### Context.filter

```ts cordis-catalog
/** Symbol key for a context's listener filter, consulted on every event dispatch. */
static readonly filter: unique symbol
```

Clave de símbolo del filtro de listeners de un contexto, consultada en cada dispatch de evento.

[Source](../../vendor/cordis/src/context.ts#L46)

### Context.isolate

```ts cordis-catalog
/** Symbol key of the isolation map (see the `Context[symbols.isolate]` property). */
static readonly isolate: unique symbol
```

Clave de símbolo del mapa de aislamiento (consulta la propiedad `Context[symbols.isolate]`).

[Source](../../vendor/cordis/src/context.ts#L48)

### Context.intercept

```ts cordis-catalog
/** Symbol key of the intercept map (see the `Context[symbols.intercept]` property). */
static readonly intercept: unique symbol
```

Clave de símbolo del mapa de interceptación (consulta la propiedad `Context[symbols.intercept]`).

[Source](../../vendor/cordis/src/context.ts#L50)

### Context.is(value)

```ts cordis-catalog
/**
 * Returns true for Cordis context proxies and context prototypes.
 *
 * Works across realms and across multiple copies of cordis, because the
 * brand is keyed by a global symbol rather than by `instanceof`.
 *
 * @param value — the value to test.
 * @returns `true` if `value` is a Cordis context, narrowing its type.
 */
static is(value: any): value is Context
```

Devuelve true para los proxies de contexto y los prototipos de contexto de Cordis.

Funciona entre realms y entre varias copias de cordis, porque la marca (brand) se clavea por un símbolo global en lugar de por `instanceof`.

- `value` — el valor que comprobar.

**Devuelve** `true` si `value` es un contexto de Cordis, estrechando su tipo.

[Source](../../vendor/cordis/src/context.ts#L61)

## Almacén de servicios y mixins

### ctx.get(name, strict?)

```ts cordis-catalog
/**
 * Read a service from the store without the inject requirement.
 *
 * @param name — the service name.
 * @param strict — when `true` (default), only return implementations
 * whose providing fiber is currently active.
 * @returns the service value, or `undefined` when not (yet) provided.
 */
get<K extends string & keyof this>(name: K, strict?: boolean): undefined | this[K]
get(name: string, strict?: boolean): any
```

Lee un servicio del almacén sin el requisito de inject.

- `name` — el nombre del servicio.
- `strict` — cuando es `true` (por defecto), solo devuelve implementaciones cuyo fiber proveedor está actualmente activo.

**Devuelve** el valor del servicio, o `undefined` cuando (todavía) no se ha provisto.

[Source](../../vendor/cordis/src/reflect.ts#L17)

### ctx.set(name, value)

```ts cordis-catalog
/**
 * Overwrite a provided service's value.
 *
 * Only the fiber that provided the service may set it; setting an
 * unprovided name throws.
 *
 * @param name — the service name.
 * @param value — the new service value.
 */
set<K extends string & keyof this>(name: K, value: undefined | this[K]): void
set(name: string, value: any): void
```

Sobrescribe el valor de un servicio provisto.

Solo el fiber que proveyó el servicio puede asignarlo; asignar un nombre no provisto lanza una excepción.

- `name` — el nombre del servicio.
- `value` — el nuevo valor del servicio.

[Source](../../vendor/cordis/src/reflect.ts#L29)

### ctx.provide(name, value)

```ts cordis-catalog
/**
 * Register a service implementation owned by the current fiber.
 *
 * The service becomes visible to dependents in the same isolation scope
 * once the fiber is active; it is unregistered (waking dependents) when
 * the returned disposer runs or the fiber unloads. Throws if the name is
 * already provided in this scope or declared as an accessor.
 *
 * @param name — the service name.
 * @param value — the service value.
 * @returns a disposer that unregisters the service.
 */
provide<K extends string & keyof this>(name: K, value: undefined | this[K]): () => void
provide(name: string, value?: any): () => void
```

Registra una implementación de servicio propiedad del fiber actual.

El servicio se vuelve visible para los dependientes del mismo ámbito de aislamiento cuando el fiber está activo; se desregistra (despertando a los dependientes) cuando se ejecuta el disposer devuelto o el fiber se descarga. Lanza una excepción si el nombre ya está provisto en este ámbito o se declara como accessor.

- `name` — el nombre del servicio.
- `value` — el valor del servicio.

**Devuelve** un disposer que desregistra el servicio.

[Source](../../vendor/cordis/src/reflect.ts#L44)

### ctx.accessor(name, options)

```ts cordis-catalog
/**
 * Define a computed context property backed by get/set hooks.
 *
 * The accessor is removed when the current fiber unloads. Throws if the
 * name is already declared.
 *
 * @param name — the context property name.
 * @param options — the `get` hook and optional `set` hook.
 */
accessor(name: string, options: Omit<Property.Accessor, 'type'>): void
```

Define una propiedad de contexto computada respaldada por hooks get/set.

El accessor se elimina cuando el fiber actual se descarga. Lanza una excepción si el nombre ya está declarado.

- `name` — el nombre de la propiedad del contexto.
- `options` — el hook `get` y el hook `set` opcional.

[Source](../../vendor/cordis/src/reflect.ts#L56)

### ctx.mixin(name, mixins)

```ts cordis-catalog
/**
 * Expose selected members of a service directly on `ctx`.
 *
 * Each mixed-in key becomes an accessor that forwards to the service
 * (binding methods to it), so e.g. `ctx.on` forwards to `ctx.events.on`.
 * Mixins are removed when the current fiber unloads.
 *
 * @param name — the context property holding the source service.
 * @param mixins — keys to forward, or a source-key → ctx-key map.
 */
mixin<K extends string & keyof this>(name: K, mixins: (keyof this & keyof this[K])[] | Dict<string>): void
mixin<T extends {}>(source: T, mixins: (keyof this & keyof T)[] | Dict<string>): void
```

Expone miembros seleccionados de un servicio directamente en `ctx`.

Cada clave mezclada se convierte en un accessor que reenvía al servicio (enlazando los métodos a él), de modo que, p. ej., `ctx.on` reenvía a `ctx.events.on`. Los mixins se eliminan cuando el fiber actual se descarga.

- `name` — la propiedad del contexto que contiene el servicio de origen.
- `mixins` — claves que reenviar, o un mapa clave-de-origen → clave-de-ctx.

[Source](../../vendor/cordis/src/reflect.ts#L67)
