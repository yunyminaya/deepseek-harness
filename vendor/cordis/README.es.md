# Cordis

[English](README.md) | Español

Cordis es un framework de plugins TypeScript para aplicaciones que necesitan inyección de dependencias explícita, servicios con alcance, cleanup gestionado por ciclo de vida y carga opcional dirigida por configuración. El paquete core se publica como `cordis`; los paquetes oficiales de este repositorio añaden un loader, includes de archivos de config, HMR, logging de consola, timers y scaffolding de proyectos.

## Instalación

```sh
yarn add cordis
```

Cordis es ESM-first. El repositorio se prueba en las versiones actuales de Node, y el scaffolder requiere Node 22 o más reciente.

## Inicio rápido

```ts
import { Context, Service } from 'cordis'

declare module 'cordis' {
  interface Context {
    counter: Counter
  }

  interface Events {
    'app/ready'(message: string): void
  }
}

class Counter extends Service {
  value = 0

  constructor(ctx: Context) {
    super(ctx, 'counter')
  }

  next() {
    return ++this.value
  }
}

const greeter = Object.assign((ctx: Context) => {
  ctx.on('app/ready', (message) => {
    ctx.logger.info('%s #%d', message, ctx.counter.next())
  })
}, {
  inject: ['counter'],
})

const root = new Context()
await root.plugin(Counter)
await root.plugin(greeter)

root.emit('app/ready', 'started')
await root.fiber.dispose()
```

Las piezas importantes son:

- `new Context()` crea el contenedor de dependencias raíz.
- `ctx.plugin()` arranca un plugin y devuelve un `Fiber`.
- `inject` le dice a Cordis qué servicios deben existir antes de que el plugin se ejecute.
- Los effects, los listeners de eventos y los servicios se eliminan cuando se elimina (dispose) su fiber propietario.

## Documentación

- [Tutorial: compilar un plugin](../../docs/tutorials/build-a-plugin.md)
- [Guía: ciclo de vida de plugins](../../docs/guides/plugin-lifecycle.md)
- [Guía: configuración del loader](../../docs/guides/loader-config.md)
- [Referencia de API](../../docs/api/core.md)

## Paquetes

| Paquete | Propósito |
| --- | --- |
| `cordis` | Contexto core, registro de plugins, ciclo de vida de fibers, eventos, servicios y logger. |
| `create-cordis` | Scaffolder de proyectos interactivo. |
| `@cordisjs/plugin-loader` | Árbol de plugins en runtime y servicio loader. |
| `@cordisjs/plugin-include` | Soporte de includes de archivos de config YAML/JSON para el loader. |
| `@cordisjs/plugin-group` | Grupos de plugins anidados para configs del loader. |
| `@cordisjs/plugin-hmr` | Hot module replacement para plugins gestionados por el loader. |
| `@cordisjs/plugin-logger-console` | Exporter de consola para el logger integrado. |
| `@cordisjs/plugin-timer` | Helpers de timeout, intervalo, throttle y debounce conscientes del disposal. |
| `@cordisjs/utils` | Utilidades compartidas usadas por los paquetes de Cordis. |

## Desarrollo

```sh
yarn install
yarn build
yarn test
yarn lint
```

El monorepo usa Yakumo para compilar y probar todos los paquetes. La mayoría de los ejemplos de los docs usan APIs públicas de `cordis`; los ejemplos del loader usan además `@cordisjs/plugin-loader` y `@cordisjs/plugin-include`.
