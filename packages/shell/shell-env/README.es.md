# @deepseek-ai/dsh-shell-env

[English](README.md) | Español

El plugin de entorno de shell independiente de herramientas: es dueño del registro `ctx.shellEnv` de variables `DSH_*` de confianza, por ejecución, que las herramientas de shell orientadas al modelo (`dsh-tool-bash`, `dsh-tool-pwsh`) recogen en el entorno de cada llamada de shell. Los hechos de shell integrados (`DSH_HOME`, `DSH_SHELL=1`, `DSH_SESSION_ID`) los posee el propio registro; otros plugins pueden registrar hechos enumerables adicionales con disposal acotado al efecto, y la propiedad duplicada o las claves de runtime no declaradas fallan en voz alta.

La raíz del paquete exporta el contrato de plugin de Cordis (`name`, `inject`, `Config`, `apply`) más la clase de servicio `ShellEnvRegistry` y sus tipos de contributor; los consumidores usan `ctx.shellEnv` tras cargar este plugin.

## Config

```yaml
- id: shell-env
  name: '@deepseek-ai/dsh-shell-env'
  config:
    dshHome: C:\Users\me\.dsh   # default: $DSH_HOME, then ~/.dsh
```

## Entorno gestionado

Cada llamada de shell del modelo, en primer plano o en segundo plano, recibe un entorno `DSH_*` de confianza recién recogido. `DSH_HOME` es el home absoluto del Harness resuelto por [`@deepseek-ai/dsh-home-paths`](../../util/home-paths/README.es.md) (config `dshHome`, luego `$DSH_HOME` ambiental, luego `~/.dsh`) y `DSH_SHELL=1` identifica al hijo gestionado. Las llamadas de agent reciben además `DSH_SESSION_ID=agent.session.header.id`; cuando el seam de persistencia activo localiza un artefacto JSONL también reciben `DSH_SESSION_JSONL=<ruta objetivo absoluta>`. La ruta JSONL es una pista de ubicación: puede no existir antes del primer flush ni contener el turno actual en búfer, y no es una credencial de autorización.

`ctx.shellEnv` es dueño de la recogida. Otros plugins pueden registrar un contributor con alcance de efecto, nombre estable, claves/descripciones declaradas y `resolve(execution: ToolExecution)`; la propiedad duplicada y las claves de runtime no declaradas fallan en voz alta, mientras que `list()` enumera las declaraciones sin ejecutar providers. Los integrados del harness reservan `DSH_HOME`, `DSH_SHELL` y `DSH_SESSION_ID`; el traductor de persistencia de este plugin es dueño de `DSH_SESSION_JSONL` leyendo el seam `sessionPersistence.locate()` neutral respecto al backend.

```ts
import type { Context } from '@deepseek-ai/cordis'
import type {} from '@deepseek-ai/dsh-shell-env'

export const inject = ['shellEnv']

export function apply(ctx: Context): void {
  ctx.shellEnv.register({
    name: 'deployment-region',
    variables: { DSH_DEPLOYMENT_REGION: { description: 'Current deployment region.' } },
    resolve: execution => execution.agent === undefined ? {} : { DSH_DEPLOYMENT_REGION: 'cn-north' },
  })
}
```

La superposición se calcula a partir del `ToolExecution` actual y se pasa por el canal dedicado `ShellExecRequest.dshEnv`. Los ejecutores locales eliminan todo `DSH_*` heredado antes de fusionar esa instantánea, de modo que los harnesses anidados y los agents padre/hijo concurrentes no puedan filtrar identidades obsoletas. `process.env` nunca se modifica. Las descripciones de las herramientas de shell enseñan la convención genérica `$DSH_*` en lugar de nombrar variables específicas de persistencia o añadir una sección permanente de system prompt.

## Experiencia del modelo

Indirectamente, a través de las herramientas de shell (`dsh-tool-bash`, `dsh-tool-pwsh`), que recogen la instantánea `DSH_*` gestionada por este registro en cada llamada de herramienta de shell.

#### Efecto en la caché KV

Sin invalidación directa; los consumidores con nombre son dueños de cualquier cambio de prefijo de petición.

## Limitaciones conocidas y trabajo diferido

- **`list()` enumera solo las variables declaradas por los contributors** — los integrados propiedad del registro (`DSH_HOME`, `DSH_SHELL`, `DSH_SESSION_ID`) no se incluyen, así que el código de diagnósticos, prompt o UI no debe tratar `list()` como un catálogo exhaustivo del entorno.
