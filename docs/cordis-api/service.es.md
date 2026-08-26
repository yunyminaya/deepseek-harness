<!-- El archivo en inglés lo genera scripts/gen-cordis-catalog.ts; este archivo en español es la contraparte revisada, mantenida mediante el emparejamiento bilingüe.
     Para actualizarlo, ejecuta primero `pnpm run gen-cordis-catalog` para regenerar el inglés y, a continuación, actualiza este archivo y ejecuta `pnpm run verify-translation-pairing --write docs/cordis-api/service.md` para volver a registrar el par. -->

# Service

[English](service.md) | Español

La clase base para los servicios de contexto. Una subclase cargada como plugin se registra a sí misma como `ctx.<name>`.

Clase base para los servicios que exponen una API con nombre en `ctx`.

Las subclases llaman a `super(ctx, name)` desde su constructor. El servicio se registra de inmediato y se elimina automáticamente junto con el fiber propietario.

[Fuente](../../vendor/cordis/src/service.ts#L11)

### service.name

```ts cordis-catalog
/** The service name this instance is registered under. */
public name!: string
```

El nombre de servicio bajo el que está registrada esta instancia.

[Fuente](../../vendor/cordis/src/service.ts#L30)

## Miembros estáticos

### Service.init

```ts cordis-catalog
/** Symbol key of an instance method run after construction (class plugins). */
static readonly init: unique symbol
```

Clave de símbolo de un método de instancia que se ejecuta después de la construcción (plugins de clase).

[Fuente](../../vendor/cordis/src/service.ts#L13)

### Service.check

```ts cordis-catalog
/** Symbol key of the availability predicate passed to `ctx.provide()`. */
static readonly check: unique symbol
```

Clave de símbolo del predicado de disponibilidad que se pasa a `ctx.provide()`.

[Fuente](../../vendor/cordis/src/service.ts#L15)

### Service.config

```ts cordis-catalog
/** Symbol key of the phantom intercept-config type parameter. */
static readonly config: unique symbol
```

Clave de símbolo del parámetro de tipo fantasma de intercept-config.

[Fuente](../../vendor/cordis/src/service.ts#L17)

### Service.invoke

```ts cordis-catalog
/** Symbol key of the call body making a service callable (e.g. `ctx.logger()`). */
static readonly invoke: unique symbol
```

Clave de símbolo del cuerpo de llamada que hace que un servicio sea invocable (p. ej. `ctx.logger()`).

[Fuente](../../vendor/cordis/src/service.ts#L19)

### Service.extend

```ts cordis-catalog
/** Symbol key of the helper deriving an extended service instance. */
static readonly extend: unique symbol
```

Clave de símbolo del helper que deriva una instancia de servicio extendida.

[Fuente](../../vendor/cordis/src/service.ts#L21)

### Service.tracker

```ts cordis-catalog
/** Symbol key of the tracker metadata used for context tracing. */
static readonly tracker: unique symbol
```

Clave de símbolo de los metadatos de tracker que se usan para el trazado de contexto.

[Fuente](../../vendor/cordis/src/service.ts#L23)

### Service.resolveConfig

```ts cordis-catalog
/** Symbol key of the intercept-config resolution helper below. */
static readonly resolveConfig: unique symbol
```

Clave de símbolo del helper de resolución de intercept-config que aparece a continuación.

[Fuente](../../vendor/cordis/src/service.ts#L25)
