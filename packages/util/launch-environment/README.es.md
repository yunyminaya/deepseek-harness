# dsh-launch-environment

[English](README.md) | Español

El entorno de esta ejecución como una única instantánea inmutable que recuerda **qué capa aportó cada valor**. Los consumidores resuelven los valores visibles al usuario contra ella en lugar de contra `process.env`, porque las capas no gozan de la misma confianza y una vista aplanada no puede distinguirlas.

| Capa | Id de origen | Qué es |
|---|---|---|
| Entorno de proceso heredado | `process` | Lo que pasaron el shell lanzador, el trabajo de CI o el contenedor — la intención explícita de esta ejecución |
| `<invocation cwd>/.env` | `project-env` | El proyecto en el que se lanzó el harness, en el que el producto confía para configurar su propio agent |
| `$DSH_HOME/.env` | `user-env` | Los valores por defecto a nivel de máquina del propio usuario |

Los valores también llegan a `process.env` — el árbol `--config` del usuario y las bibliotecas de terceros lo leen — pero esa vista aplanada no es la autoridad para nada de lo que el harness resuelve.

## Resolución

`get(name)` busca en todas las capas, de mayor a menor confianza. `getFrom(name, sources)` busca solo en las capas nombradas sin cambiar ese orden de confianza.

**Omitir una capa es una negativa, no una degradación** — un llamador que nunca debe aceptar una capa la deja fuera de la lista, de modo que ningún reordenamiento futuro puede hacerla volver. Los adaptadores de provider nombran las tres, porque el producto confía en el proyecto en el que se ejecuta; el mecanismo existe para las decisiones en las que eso no se cumple.

Los nombres se comparan como los compara la plataforma: exactamente en POSIX, sin distinguir mayúsculas en Windows. Una búsqueda sensible a mayúsculas allí clasificaría la capa equivocada — el `deepseek_api_key` de un shell y el `DEEPSEEK_API_KEY` de un `.env` de proyecto son una sola variable para el SO, y tratarlos como dos haría ganar al proyecto.

```ts
import type { Context } from '@deepseek-ai/cordis'
import { launchEnvironmentOf } from '@deepseek-ai/dsh-launch-environment'

declare const ctx: Context
const endpoint = launchEnvironmentOf(ctx).get('DEEPSEEK_BASE_URL')?.value
```

`launchEnvironmentOf(ctx)` devuelve la instantánea del lanzador cuando el CLI del producto arrancó el árbol y, en caso contrario, el entorno heredado como única capa. Ese fallback no debilita las reglas: un host de SDK o un `cordis.yml` desnudo no descubrió archivos, así que todo lo que tiene es realmente el entorno con el que se lanzó.

## Limitaciones conocidas y trabajo diferido

- **La instantánea no es una frontera de subprocesos** — cada capa también se materializa en `process.env`, por lo que las variables ordinarias del proyecto llegan a los procesos hijos bajo la depuración de [`dsh-subprocess`](../../subprocess/subprocess/README.md). El [contrato `.env`](../../boot/app-boot/README.es.md#profiles) del lanzador del producto rechaza las variables de arranque antes de la materialización.
- **Sin capa por workspace** — la capa de proyecto es el directorio *invocador*, fijado en el lanzamiento. Un workspace seleccionado después en la Web UI no contribuye nada, deliberadamente: seguirlo permitiría que el propio workspace de un modelo cambiara el entorno del harness a mitad de sesión.
