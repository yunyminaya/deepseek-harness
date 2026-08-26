# @cordisjs/plugin-logger-console

[English](README.md) | Español

Exporter de consola para el servicio logger integrado de Cordis.

## Uso

```ts
import { Context } from 'cordis'
import ConsoleLogger from '@cordisjs/plugin-logger-console'

const root = new Context()
await root.plugin(ConsoleLogger, {
  showDiff: true,
  levels: {
    default: 2,
    hmr: 3,
  },
})

root.logger('app').info('started')
```

## Config

| Campo | Descripción |
| --- | --- |
| `colors` | Nivel de soporte de color, o `false` para deshabilitar los colores. |
| `maxLength` | Longitud máxima de línea renderizada antes del truncado. |
| `levels` | Mapa de nivel mínimo por logger. |
| `showDiff` | Muestra el tiempo transcurrido desde el mensaje anterior. |
| `showTime` | Plantilla de timestamp. |
| `label` | Opciones de ancho, margen y alineación de la etiqueta. |

La entrada Node usa `node:util.inspect` para `%o` y `%O`; la entrada de navegador pasa los argumentos de log a `console`.
