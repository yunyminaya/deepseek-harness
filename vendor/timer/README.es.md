# @cordisjs/plugin-timer

[English](README.md) | Español

Servicio de temporizadores consciente de la disposición para Cordis.

## Uso

```ts
import { Context } from 'cordis'
import Timer from '@cordisjs/plugin-timer'

const root = new Context()
await root.plugin(Timer)

const dispose = root.timeout(() => {
  root.logger.info('done')
}, 1000)

dispose()
```

Los handles de Timer se registran en la fiber actual, por lo que se limpian automáticamente cuando se dispone el plugin que los creó.

## API

| API | Descripción |
| --- | --- |
| `ctx.timeout(callback, delay)` | Ejecuta una vez y devuelve un disposer. |
| `ctx.timeout(delay)` | Devuelve una promesa que se resuelve después de `delay`. |
| `ctx.interval(callback, delay)` | Ejecuta repetidamente y devuelve un disposer. |
| `ctx.interval(delay)` | Devuelve un iterador asíncrono que produce un valor en cada intervalo. |
| `ctx.throttle(callback, delay, noTrailing?)` | Devuelve una función con throttling con `.dispose()`. |
| `ctx.debounce(callback, delay)` | Devuelve una función con debounce con `.dispose()`. |

`ctx.setTimeout()` y `ctx.setInterval()` se conservan como alias obsoletos de `ctx.timeout()` y `ctx.interval()`.
