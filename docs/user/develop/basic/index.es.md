# Tu primer plugin

[English](index.md) | Español

Este tutorial crea un plugin mínimo de Harness y lo carga en la Web UI. Parte de un checkout del repositorio que haya completado la [ruta de ejecución desde el código fuente](../../../../README.es.md#run-from-source).

## Crea un proyecto local

Desde la raíz del repositorio, crea un proyecto de prueba para el tutorial:

```sh
mkdir -p scratch-plugin/src
```

## ¿Qué es un plugin?

En Harness, un plugin es un módulo de TypeScript que exporta una función `apply`. El framework llama a `apply` al cargar el plugin y le pasa un objeto de contexto `ctx` a través del cual el plugin registra capacidades:

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'my-plugin'

export function apply(ctx: Context) {
  // Register capabilities here.
}
```

Eso es toda la configuración.

## Crea el archivo del plugin

Crea `scratch-plugin/src/my-plugin.ts`:

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'hello-plugin'

export function apply(ctx: Context) {
  // Required dependencies are ready before apply runs.
  console.log('[hello-plugin] plugin loaded!')
}
```

## Regístralo en cordis.yml

Ejecuta `pwd` desde la raíz del repositorio y crea después `scratch-plugin/cordis.yml` como un overlay de Web que inserta el plugin local. Reemplaza `/absolute/path/to/deepseek-harness` de abajo por la ruta impresa:

```yaml
- insert:
    - id: hello
      name: '/absolute/path/to/deepseek-harness/scratch-plugin/src/my-plugin.ts'
```

La ruta del plugin debe ser absoluta. Un archivo patch aporta configuración pero no cambia el directorio de profile desde el que el loader resuelve las rutas de los módulos.

Inicia la Web UI con ese overlay:

```sh
pnpm dsh web --patch ./scratch-plugin/cordis.yml
```

Abre `http://127.0.0.1:3080`. El terminal imprime `[hello-plugin] plugin loaded!` durante el arranque.

## Limpieza automática

Todo lo que se registra a través de `ctx` —listeners de eventos, herramientas o temporizadores— se limpia cuando el plugin se descarga. No necesitas llamar a removeListener ni a clearInterval manualmente.

Para un recurso que necesita una limpieza explícita, como una conexión de red, usa `ctx.effect()` para proporcionar su disposer:

```ts
import type { Context } from '@deepseek-ai/cordis'

export function apply(ctx: Context) {
  ctx.effect(() => {
    const timer = setInterval(() => {
      console.log('heartbeat')
    }, 5000)

    // The returned function runs when the plugin unloads.
    return () => clearInterval(timer)
  })
}
```

## Declara dependencias

Si el plugin consume otro servicio como `tools` o `llm`, decláralo en `inject`:

```ts ignore-check
import type { Context } from '@deepseek-ai/cordis'

export const name = 'my-tool-plugin'
export const inject = ['tools']

export function apply(ctx: Context) {
  // ctx.tools is ready here.
  ctx.tools.register(/* ... */)
}
```

El framework espera a que todos los servicios requeridos estén listos antes de cargar el plugin.

## Tres formas de plugin

Además de un módulo con función, un plugin puede usar la forma de objeto o de clase.

### Forma de objeto

```ts
import type { Context } from '@deepseek-ai/cordis'

export default {
  name: 'my-plugin',
  inject: ['tools'],
  apply(ctx: Context) {
    // ...
  },
}
```

### Forma de clase

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

export default class MyService extends Service {
  static inject = ['tools']

  constructor(ctx: Context) {
    super(ctx, 'myService')
    // Perform synchronous initialization in the constructor.
  }
}
```

La forma de función es suficiente en la mayoría de los casos. Usa la forma de clase cuando el plugin proporcione un servicio a otros plugins; consulta [servicios y dependencias](../framework/service.es.md).

## Pasos siguientes

- [Crea una herramienta](./tool.es.md) — aprende el DSL de definición de herramientas
- [Configuración de plugin](./config.es.md) — acepta la configuración del usuario
- [Tutorial de Cordis](../../../cordis-tutorial/index.es.md) — el framework de plugins subyacente, construido desde un directorio de prueba sin clave de API
