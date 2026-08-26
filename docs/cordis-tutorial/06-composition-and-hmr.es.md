# 6. Composición y HMR (hot module replacement)

[English](06-composition-and-hmr.md) | Español

Toda capacidad construida hasta ahora es un plugin, y `cordis.yml` selecciona el árbol de plugins de la aplicación. Este capítulo cambia esa composición, recarga un plugin en caliente y diagnostica un plugin que nunca llega a cargarse.

## Las entradas son más que un nombre

Una entrada de config acepta metadatos más allá de `name` y `config`:

```yaml
- id: greeter          # stable identity for this entry
  name: './greeter.ts'
- id: consumer
  name: './consumer.ts'
  disabled: true       # keep the entry, skip mounting it
```

`id` le da a la entrada una identidad estable para que el loader distinga una edición de una entrada existente de una eliminación más una adición. `disabled: true` desmonta un plugin sin borrar su entrada: dale la vuelta al valor y el plugin (y todo lo que esté PENDING en sus servicios) se carga de nuevo.

Los grupos anidan una sublista de entradas que se cargan y descargan como una unidad, e `isolate` le da a un grupo su propia instancia de un nombre de servicio: dos grupos pueden ver cada uno un provider de `shell` configurado de forma distinta sin afectarse entre sí. El [primer de Cordis](../cordis-primer.es.md) y el [ejemplo de aislamiento de servicios](../user/develop/framework/service.es.md#service-isolation) cubren los detalles.

## Reemplazo de módulos en caliente

Como descargar libera los efectos ([capítulo 2](02-lifecycle-and-effects.es.md)) y cargar sigue las dependencias ([capítulo 3](03-services.es.md)), el HMR puede reemplazar un plugin en ejecución descargándolo y cargándolo. El plugin `@deepseek-ai/cordis-plugin-hmr` vigila tus archivos y hace exactamente eso al guardar.

En `tmp/cordis-tutorial`, escribe `cordis.yml`:

```yaml
- id: logger
  name: '@deepseek-ai/cordis-plugin-logger-console'
- id: timer
  name: '@deepseek-ai/cordis-plugin-timer'
- id: hmr
  name: '@deepseek-ai/cordis-plugin-hmr'
  config:
    root: ['.']
- id: hello
  name: './hello.ts'
```

Dos plugins de soporte se unieron a la lista: el HMR registra a través del servicio logger de Cordis, así que sin un exporter de consola no verías sus mensajes, y hace `inject` del servicio `timer` para el debounce: sin `@deepseek-ai/cordis-plugin-timer` se queda en PENDING para siempre, en silencio. Ese silencio es el tema de la siguiente sección.

El HMR lee los internals del loader de Node a través del helper nativo del Loader. Ejecuta Cordis bajo tsx:

```sh
node --import tsx ../../vendor/cordis/bin.js
```

Ahora edita `hello.ts` —cambia el mensaje de log— y guarda:

```
hello from my first plugin
2026-07-22 15:44:36 [I] hmr watching [ '.' ]
2026-07-22 15:44:39 [I] hmr reload plugin at hello.ts
hello from my EDITED plugin
```

La instancia anterior se descargó (todos sus efectos se deshicieron), el código nuevo se cargó y `apply` volvió a ejecutarse. Detén el proceso con Ctrl-C. Editar el propio `cordis.yml` también se detecta: el loader hace diff de las entradas por `id` y monta, desmonta o reconfigura solo lo que cambió. Por eso las entradas de arriba llevan `id` explícitos: una entrada sin id recibe un id generado en cada lectura, así que tras cualquier edición del archivo de config cuenta como eliminada-más-añadida y se remonta aunque sus propias líneas no hayan cambiado.

## Diagnosticar un plugin que nunca se carga

La cara opuesta de la carga impulsada por dependencias: un plugin cuyo `inject` nombra un servicio que nadie provee espera para siempre sin imprimir nada. Sin error: PENDING es un estado legítimo, porque el provider puede montarse más tarde.

Puedes ver los estados directamente. Todo context puede enumerar el registro de plugins; crea `diagnose.ts`:

```ts
import { FiberState, type Context } from '@deepseek-ai/cordis'

export const name = 'diagnose'

export function apply(ctx: Context) {
  setTimeout(() => {
    for (const runtime of ctx.registry.values()) {
      for (const fiber of runtime.fibers) {
        if (fiber.state === FiberState.PENDING) {
          console.log(`${fiber.name} is PENDING — a required service is missing`)
        }
      }
    }
  }, 500)
}
```

Y un plugin con una dependencia insatisfacible, `needs-timer.ts`:

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'needs-timer'
export const inject = ['timer']

export function apply(ctx: Context) {
  console.log('needs-timer loaded')
}
```

```yaml
- name: './needs-timer.ts'
- name: './diagnose.ts'
```

Ejecútalo (`node --import tsx ../../vendor/cordis/bin.js` sin más; detén con Ctrl-C):

```
needs-timer is PENDING — a required service is missing
```

`inject: ['timer']` no tiene provider. Añade `- name: '@deepseek-ai/cordis-plugin-timer'` a la lista y el plugin se carga. Cuando un plugin no hace nada y no reporta nada, inspecciona su estado de fiber. Iterar sin el filtro de PENDING también muestra los plugins propios del loader (Loader, Include) como fibers ACTIVE, porque los plugins montan el propio archivo de config.

Siguiente: [Dentro del harness](07-into-the-harness.es.md) — los mismos patrones contra servicios reales del harness.

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)
