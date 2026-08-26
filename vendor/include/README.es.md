# @cordisjs/plugin-include

[English](README.md) | Español

Árbol del loader respaldado por archivos para Cordis. El plugin include lee un archivo YAML o JSON, lo convierte en entradas del loader y escribe las actualizaciones de vuelta cuando el archivo es escribible.

## Uso

```ts
import { Context } from 'cordis'
import Loader from '@cordisjs/plugin-loader'
import Include from '@cordisjs/plugin-include'

const root = new Context()
await root.plugin(Loader, { baseUrl: import.meta.url })
await root.plugin(Include, {
  path: './cordis.yml',
  initial: [],
  enableLogs: true,
})
```

Ejemplo de `cordis.yml`:

```yaml
- id: timer
  name: '@cordisjs/plugin-timer'
- id: app
  name: ./plugins/app
  config:
    message: hello
```

## Config

| Campo | Descripción |
| --- | --- |
| `path` | Ruta del archivo YAML o JSON resuelta desde `ctx.baseUrl`. |
| `initial` | Lista de entradas escrita cuando el archivo falta. |
| `patches` | Patches de runtime aplicados después de leer el archivo. |
| `enableLogs` | Habilita los logs de apply, recarga y unload del loader. |

Los patches pueden insertar entradas o sobrescribir campos en entradas con un `id` coincidente.
