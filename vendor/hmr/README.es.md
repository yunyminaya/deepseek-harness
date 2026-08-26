# @cordisjs/plugin-hmr

[English](README.md) | Español

Hot module replacement para plugins de Cordis gestionados por el loader.

El plugin de HMR observa los archivos de fuente, rastrea el grafo de módulos de Node, limpia las cachés de módulos afectadas y recarga solo las entradas de plugins que dependen de los archivos de aplicación cambiados. Los cambios en dependencias de nivel de framework caen a `loader.exit()`, dejando que el proceso host se reinicie.

Los watchers de módulos canonizan su directorio base existente antes de abrir Chokidar. Los watchers de config exactos canonizan igualmente el ancestro existente más profundo y restauran después cualquier sufijo ausente. Los callbacks y los diagnósticos conservan el nombre de archivo absoluto solicitado, mientras que el backend nativo recibe una única grafía del sistema de archivos incluso cuando Windows suministró un alias 8.3.

## Requisitos

- `@cordisjs/plugin-loader`
- `@cordisjs/plugin-timer`
- Un runtime que exponga el loader de módulos interno de Node. El paquete lanza una excepción si el servicio loader no tiene un loader de módulos interno disponible.

## Uso

```yaml
- id: timer
  name: '@cordisjs/plugin-timer'
- id: hmr
  name: '@cordisjs/plugin-hmr'
  config:
    root:
      - src
    ignored:
      - '**/node_modules'
      - '**/.*'
    debounce: 100
```

## Config

| Campo | Descripción |
| --- | --- |
| `base` | Directorio base opcional resuelto desde `ctx.baseUrl`. |
| `root` | Raíces de Chokidar a observar. Por defecto `['.']`. |
| `ignored` | Patrones de Picomatch excluidos del análisis de observación y recarga. |
| `debounce` | Milisegundos a esperar antes de procesar una ráfaga de cambios. |

## Eventos

| Evento | Descripción |
| --- | --- |
| `hmr/change` | Se emite para los archivos cambiados que no gestionan ni la recarga de plugins ni la recarga de config. |
| `hmr/reload` | Se emite después de recargar una o más entradas de plugins. |
