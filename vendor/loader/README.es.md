# @cordisjs/plugin-loader

[English](README.md) | Español

Loader de plugins en runtime para Cordis. El loader es dueño de un `EntryTree`, importa los módulos de plugins por nombre, aplica su config y mantiene el grafo de plugins en ejecución sincronizado con las actualizaciones de entradas.

## Uso

```ts
import { Context } from 'cordis'
import Loader from '@cordisjs/plugin-loader'

const root = new Context()
await root.plugin(Loader, { baseUrl: import.meta.url })

const id = await root.loader.create({
  name: './plugins/example',
  config: { enabled: true },
})

await root.loader.await()
root.loader.update(id, { config: { enabled: false } })
```

## Opciones de entrada

| Campo | Descripción |
| --- | --- |
| `id` | Id estable para resolver, actualizar y eliminar la entrada. |
| `name` | Especificador de módulo importado por el loader. |
| `config` | Config pasada al plugin. |
| `group` | Marca la entrada como un grupo cuyo `config` es una lista de entradas hijas. |
| `disabled` | Detiene la entrada e impide que se inicie. |
| `inject` | Añade servicios requeridos o intercepta la config de esta entrada. |

## API

| API | Descripción |
| --- | --- |
| `loader.create(options, parent?, position?)` | Añade y arranca una entrada. |
| `loader.update(id, options, parent?, position?)` | Actualiza, mueve y reinicia una entrada. |
| `loader.remove(id)` | Detiene y elimina una entrada. |
| `loader.resolve(id)` | Resuelve una entrada por id, incluidos los ids anidados `a:b`. |
| `loader.resolveGroup(id)` | Resuelve el grupo raíz o un grupo anidado. |
| `loader.await()` | Espera los imports de entradas pendientes y las recargas de fibers. |
| `loader.locate(fiber?)` | Devuelve el id de entrada del loader propietario de una fiber. |

Para árboles respaldados por archivos, usa `@cordisjs/plugin-include`.
