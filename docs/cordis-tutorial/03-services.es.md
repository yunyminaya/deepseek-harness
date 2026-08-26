# 3. Servicios

[English](03-services.md) | Español

Un **servicio** es una capacidad con nombre que un plugin provee y que otros plugins consumen a través de `ctx`. En el harness, `ctx.tools`, `ctx.llm` y `ctx.agents` son servicios. Un consumidor nombra la capacidad, como `'tools'`, en lugar de importar a su provider; así, la configuración puede elegir un provider sin cambiar al consumidor.

## Provee un servicio

Crea `greeter.ts` en `tmp/cordis-tutorial`:

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Context {
    greeter: GreeterService
  }
}

export class GreeterService extends Service {
  constructor(ctx: Context) {
    super(ctx, 'greeter')
  }

  greet(who: string) {
    return `Hello, ${who}!`
  }
}

export const name = 'greeter'

export function apply(ctx: Context) {
  ctx.plugin(GreeterService)
}
```

Dos piezas trabajan juntas:

- **En runtime**: `super(ctx, 'greeter')` registra la instancia bajo el nombre `greeter`. A partir de ahí, cualquier plugin puede acceder a ella como `ctx.greeter`. El registro es un efecto: descargar al provider elimina el servicio.
- **En tiempo de compilación**: el bloque `declare module '@deepseek-ai/cordis'` es fusión de declaraciones de TypeScript. Añade `greeter` a la interfaz `Context` para que `ctx.greeter` pase el typecheck en todas partes. No genera código; sin él, el servicio sigue funcionando en runtime, pero los consumidores pierden la seguridad de tipos.

Una subclase de `Service` es en sí misma un plugin (la forma de clase del capítulo 1), así que `ctx.plugin(GreeterService)` la monta como a cualquier otro.

## Consume un servicio con `inject`

Crea `consumer.ts`:

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'consumer'
export const inject = ['greeter']

export function apply(ctx: Context) {
  console.log(ctx.greeter.greet('world'))
}
```

`inject` enumera los servicios que requiere este plugin. Cordis mantiene el plugin en PENDING hasta que existe cada servicio enumerado, así que dentro de `apply` tienes garantizado que `ctx.greeter` está listo. El orden de carga en `cordis.yml` no importa: las dependencias, y no el orden del archivo, deciden cuándo arrancan los plugins.

Compón y ejecuta:

```yaml
- name: './greeter.ts'
- name: './consumer.ts'
```

```
Hello, world!
```

Intercambia las dos líneas de `cordis.yml` y vuelve a ejecutar: la misma salida. Prueba a eliminar `./greeter.ts` por completo: el consumidor permanece en PENDING y no imprime nada — ni fallo ni ejecución parcial. Un fiber en PENDING tampoco mantiene vivo el event loop de Node, así que una composición sin nada más en ejecución termina con 0 en silencio. El [capítulo 6](06-composition-and-hmr.es.md) muestra cómo diagnosticar ese estado.

## Las dependencias se rastrean después de la carga

`inject` no es una comprobación de arranque de una sola vez. Si un servicio requerido desaparece mientras la app se ejecuta — su provider se descargó o fue sustituido en caliente — también se descargan todos los plugins dependientes, y vuelven a cargarse cuando el servicio regresa. Combinado con los efectos ([capítulo 2](02-lifecycle-and-effects.es.md)), esto evita que un consumidor en ejecución conserve una referencia a un servicio no disponible: sus propios registros se deshacen cuando la dependencia desaparece.

Esta es también la razón por la que la sustitución de servicios funciona en la configuración: descarga la entrada `dsh-bash-local`, monta un provider `shell` distinto, y todos los plugins que inyectan `'shell'` se reinician limpiamente contra la nueva implementación.

## Dependencias opcionales

`inject` es para requisitos estrictos. Para una capacidad de la que el plugin puede prescindir, omite `inject` y sondea en el lugar de uso:

```ts ignore-check
export function apply(ctx: Context) {
  // undefined when no provider is loaded; the plugin still runs.
  const greeter = ctx.get('greeter')
  console.log(greeter?.greet('maybe') ?? 'no greeter available')
}
```

## Nombres

Los nombres de servicio viven en un único namespace plano por aplicación. Pon prefijo o namespace a tus propios servicios de forma distintiva (el harness reclama nombres simples como `tools` y `llm`); las regiones `cordis-surface` generadas en las [páginas de subsistemas](../subsystems/core.es.md) enumeran todos los nombres que registra el harness.

Siguiente: [Eventos](04-events.es.md) — comunicación sin un servicio compartido.

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)
