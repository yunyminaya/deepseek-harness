# 2. Ciclo de vida y efectos

[English](02-lifecycle-and-effects.md) | Español

Un plugin de Cordis puede descargarse por una edición de la configuración, una recarga en caliente, una liberación explícita o la pérdida de un servicio requerido. Los registros hechos a través de las API de Cordis son efectos y se deshacen cuando se descarga el plugin que los posee; los recursos gestionados fuera de esas API deben envolverse en `ctx.effect()`.

## Efectos

Para un recurso que Cordis aún no gestiona — un temporizador, una conexión, un watcher — envuélvelo en `ctx.effect()` y devuelve un disposer:

Crea `lifecycle.ts` en `tmp/cordis-tutorial`:

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'lifecycle-demo'

function heartbeat(ctx: Context) {
  console.log('heartbeat plugin loading')
  ctx.effect(() => {
    const timer = setInterval(() => console.log('tick'), 200)
    return () => {
      clearInterval(timer)
      console.log('heartbeat cleaned up')
    }
  })
}

export function apply(ctx: Context) {
  // Mount a child plugin and keep its fiber to dispose it later.
  const fiber = ctx.plugin(heartbeat)
  // The demo timer is itself an effect: if THIS plugin is unloaded first,
  // the pending callback is cancelled instead of firing on a dead app.
  ctx.effect(() => {
    const timer = setTimeout(async () => {
      await fiber.dispose()
      console.log('disposed')
      process.exit(0)
    }, 700)
    return () => clearTimeout(timer)
  })
}
```

Apunta `cordis.yml` a él:

```yaml
- name: './lifecycle.ts'
```

Ejecuta (`node --import tsx ../../vendor/cordis/bin.js`) y obtienes:

```
heartbeat plugin loading
tick
tick
tick
heartbeat cleaned up
disposed
```

Tres cosas que debes notar:

- `ctx.plugin(heartbeat)` monta una función **desde código** como plugin — la misma operación que el loader de YAML realiza para cada entrada de configuración. Un plugin función no necesita método `apply`: Cordis llama a la función directamente y usa su nombre solo para los diagnósticos. El método `apply` solo es necesario en la forma de objeto, `ctx.plugin({ apply(ctx) { /* ... */ } })`. La llamada devuelve un **fiber**, el manejador de runtime de una instancia de plugin cargada.
- El cuerpo del efecto se ejecuta durante la carga; el disposer que devuelve se ejecuta durante la descarga. Nunca llamas tú al disposer para un recurso que vive lo que el plugin.
- `fiber.dispose()` se resuelve cuando ha terminado toda la limpieza del plugin — incluidos los disposers asíncronos — y descarga de forma recursiva cualquier plugin hijo que haya montado.

## La máquina de estados del fiber

Cada instancia de plugin cargada posee un fiber que recorre estos estados:

```
PENDING → LOADING → ACTIVE → UNLOADING → DISPOSED
                 ↘ FAILED
```

- **PENDING** — declarado, pero un servicio requerido (capítulo 3) aún no está disponible.
- **LOADING / ACTIVE** — `apply` se está ejecutando / ha terminado.
- **FAILED** — `apply` o la validación de configuración lanzó una excepción.
- **UNLOADING / DISPOSED** — los disposers se están ejecutando / todo está desmontado.

Volverás a encontrarte con PENDING en el [capítulo 6](06-composition-and-hmr.es.md), donde suele ser la respuesta a «¿por qué mi plugin no imprime nada?».

## Qué es ya un efecto

Rara vez escribes `ctx.effect()` tú mismo, porque las API de registro integradas ya son efectos:

- `ctx.on(event, listener)` — el listener se elimina al descargar ([capítulo 4](04-events.es.md)).
- `ctx.plugin(child)` — el hijo se libera junto con su padre.
- Los registros de servicios son efectos. Los registros del harness, como `ctx.tools.register(...)`, también adjuntan los disposers que devuelven al plugin que los llama, de modo que se deshacen automáticamente ([capítulo 7](07-into-the-harness.es.md)).

Para un recurso que Cordis no gestiona, adquiérelo dentro de `ctx.effect()` y devuelve un disposer que lo libere. Cordis invoca entonces esa liberación durante la descarga, incluida la recarga en caliente.

Una salvedad sobre el orden: los disposers arrancan en orden inverso al de registro, pero varios disposers **asíncronos** se ejecutan de forma concurrente. Si los pasos de desmontaje deben ejecutarse en secuencia, mantenlos en un único disposer y espéralos allí.

Siguiente: [Servicios](03-services.es.md) — cómo comparten capacidades los plugins.

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)
