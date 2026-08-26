# Tutorial de Cordis

[English](index.md) | Español

Cordis es el framework de plugins sobre el que se asienta DeepSeek Harness: un runtime pequeño donde cada capacidad —herramientas, adaptadores de LLM (modelo de lenguaje de gran tamaño), acceso a archivos, el propio agent loop (bucle del agente)— es un plugin montado en un context compartido. Este tutorial enseña Cordis en la práctica: cada capítulo es un ejemplo ejecutable que construyes en un directorio de pruebas dentro de este repositorio, y termina con un plugin conectado a servicios reales del harness.

El público son los agent developers. No necesitas experiencia profunda en TypeScript; las [notas de TypeScript](#typescript-notes) de abajo explican la sintaxis que pueda resultarte desconocida, y cada capítulo muestra los comandos exactos y la salida esperada.

Si prefieres la referencia condensada de conceptos en lugar de un recorrido guiado, lee el [primer de Cordis](../cordis-primer.es.md). La referencia exhaustiva de la API vive en las regiones generadas `cordis-surface` de las [páginas de subsistemas](../subsystems/core.es.md) y en las páginas de la [API core de Cordis](../cordis-api/context.es.md).

Para escribir plugins para el propio harness —cargados desde un `cordis.yml` y manejados desde la Web UI en lugar del launcher de abajo—, empieza por [tu primer plugin de Harness](../user/develop/basic/index.es.md).

## Configuración

Necesitas un clon de este repositorio con las dependencias instaladas; la [guía de desarrollo](../development.es.md#setup-tutorial) enumera los prerrequisitos. Este tutorial no necesita ninguna API key; todos los ejemplos se ejecutan sin clave.

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
pnpm install
```

Crea el directorio de pruebas en el que trabajan los capítulos. `tmp/` está en gitignore, así que nada de lo que escribas allí toca el control de versiones:

```sh
mkdir -p tmp/cordis-tutorial
cd tmp/cordis-tutorial
```

Todos los capítulos ejecutan el mismo comando desde este directorio:

```sh
node --import tsx ../../vendor/cordis/bin.js
```

Ese launcher de un solo archivo (consulta [vendor/cordis/bin.js](../../vendor/cordis/bin.js)) crea un `Context` raíz, monta el plugin Loader y le dice que cargue `./cordis.yml` desde el directorio actual. Todo lo demás —qué plugins existen, cómo están configurados— viene de ese archivo YAML, que escribirás dentro de un momento. La bandera `--import tsx` permite que Node ejecute los archivos TypeScript a los que apunta la config sin un paso de build.

## Capítulos

1. [Tu primer plugin](01-first-plugin.es.md) — un plugin es una función; el loader lo monta.
2. [Ciclo de vida y efectos](02-lifecycle-and-effects.es.md) — los registros gestionados por Cordis se deshacen cuando su plugin se descarga.
3. [Servicios](03-services.es.md) — expón una capacidad en `ctx` y depende de ella con `inject`.
4. [Eventos](04-events.es.md) — eventos tipados, despacho por difusión y el short-circuit del waterfall (cascada de eventos).
5. [Configuración](05-config.es.md) — config validada desde `cordis.yml`, que falla ruidosamente con entradas malas.
6. [Composición y HMR (hot module replacement)](06-composition-and-hmr.es.md) — el archivo de config como árbol de plugins, recarga en caliente y diagnóstico de un plugin que nunca se carga.
7. [Dentro del harness](07-into-the-harness.es.md) — registra una herramienta invocable por el modelo contra servicios reales del harness.

<a id="typescript-notes"></a>

## Notas de TypeScript

Los ejemplos usan tres funcionalidades de TypeScript más allá del JavaScript moderno ordinario:

- **Las anotaciones de tipo** describen valores sin cambiar el comportamiento en runtime: `ctx: Context` dice que `ctx` tiene la API de context de Cordis, `who: string` acepta texto y `string[]` significa un array de strings.
- **`import type { Context } from '@deepseek-ai/cordis'`** importa solo información de tipos. Desaparece en runtime, así que un archivo de plugin que necesita `Context` únicamente para anotaciones no añade ninguna dependencia en runtime.
- **El declaration merging** (`declare module '@deepseek-ai/cordis' { ... }`) añade tus entradas a interfaces que Cordis ya declara —por ejemplo, el tipo de una nueva propiedad `ctx.greeter` o un nombre de evento. No genera ningún cableado en runtime; el plugin provee el servicio o emite el evento por separado. El capítulo 3 muestra el patrón completo.

El capítulo 5 también usa una `interface` para describir los campos de un objeto de configuración y un tipo genérico como `Schema<Config>` para decir qué campos de un objeto valida un schema. Puedes copiar esas declaraciones tal cual; el texto que las rodea explica qué conecta cada una.

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)
