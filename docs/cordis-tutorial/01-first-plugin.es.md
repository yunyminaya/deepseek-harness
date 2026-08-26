# 1. Tu primer plugin

[English](01-first-plugin.md) | Español

En la configuración de loader que se usa aquí, un módulo de plugin de Cordis exporta por nombre una función `apply`. Cuando Cordis lo carga, llama a `apply` con un **contexto** — el objeto `ctx` a través del cual el plugin registra todo lo que aporta.

## Escribe el plugin

En tu directorio `tmp/cordis-tutorial` (consulta [la preparación](index.es.md#setup)), crea `hello.ts`:

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'hello'

export function apply(ctx: Context) {
  console.log('hello from my first plugin')
}
```

La exportación `name` es metadato de visualización opcional; etiqueta el plugin en los diagnósticos.

## Compón la aplicación

El lanzador de este tutorial ensambla la aplicación a partir de la configuración. Crea `cordis.yml`:

```yaml
- name: './hello.ts'
```

El archivo es una lista de entradas de plugin. `name` es un especificador de módulo — una ruta relativa o un nombre de paquete npm — y el loader monta cada entrada. Las entradas arrancan de forma concurrente, así que la posición en la lista no garantiza qué plugin se carga primero; el orden viene de las dependencias de servicios (`inject`, [capítulo 3](03-services.es.md)), no de la posición en el archivo.

## Ejecútalo

```sh
node --import tsx ../../vendor/cordis/bin.js
```

Salida esperada:

```
hello from my first plugin
```

El proceso termina por sí solo cuando no queda nada en ejecución. Qué ha pasado:

1. El lanzador creó un `Context` raíz y montó el plugin **Loader**.
2. El Loader leyó `cordis.yml`, resolvió `./hello.ts` y lo montó como plugin hijo.
3. Cordis llamó a tu `apply(ctx)`.

En tu archivo no hay código de arranque del framework: un plugin describe lo que aporta y `cordis.yml` compone la aplicación. La [base de `dsh`](../../packages/bundle/base/cordis.patch.yml), por ejemplo, es una composición de plugins más larga que los overlays de despliegue parchean.

## Las otras dos formas de plugin

La función es la forma más común, pero Cordis acepta tres:

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

// 1. Function plugin (what you just wrote).
export function apply(ctx: Context) {}

// 2. Object plugin: an object with an `apply` method.
export const objectPlugin = {
  name: 'object-plugin',
  apply(ctx: Context) {},
}

// 3. Class plugin: a Service subclass (covered in chapter 3).
export class MyService extends Service {
  constructor(ctx: Context) {
    super(ctx, 'myTutorialService')
  }
}
```

Usa la forma de función hasta que necesites exponer un servicio; el [capítulo 3](03-services.es.md) explica cuándo la forma de clase se gana su lugar.

## Prueba a romperlo

Haz que `apply` lance una excepción:

```ts ignore-check
export function apply(ctx: Context) {
  throw new Error('apply exploded')
}
```

Ejecútalo de nuevo: el proceso muere con tu error. Un plugin que falla al cargar es un fallo ruidoso, no una entrada omitida.

Una advertencia que conviene conocer pronto: una entrada de configuración cuyo módulo no puede **resolverse** — una ruta o un nombre de paquete mal escritos — se notifica a través del servicio de logger de Cordis en lugar de hacer fallar el proceso, y durante el arranque esa notificación puede perderse antes de que un exportador de consola esté escuchando. Si una entrada recién añadida parece no hacer nada, revisa primero la ortografía.

Siguiente: [Ciclo de vida y efectos](02-lifecycle-and-effects.es.md) — qué ocurre cuando un plugin se descarga.

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)
