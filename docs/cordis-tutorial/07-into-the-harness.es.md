# 7. Dentro del harness

[English](07-into-the-harness.md) | Español

Este capítulo registra una herramienta invocable por el modelo en el servicio `tools` del harness, la ejecuta a través del pipeline de herramientas del harness y observa el evento de resultado. Sigue sin clave y no llama a ningún modelo.

## Un plugin de herramienta

Crea `greet-tool.ts` en `tmp/cordis-tutorial`:

```ts
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'
import { CallId } from '@deepseek-ai/dsh-llm'

export const name = 'greet-tool'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'greet',
    description: 'Greet the named person.',
    parameters: {
      name: { type: 'string', required: true, description: 'Who to greet' },
    },
    output: {
      schema: { type: 'string' },
      render: (_args, value) => [{ type: 'text', text: value }],
    },
    async execute(args) {
      return `Hello, ${args.name}!`
    },
  }))

  // Drive one call through the real execution pipeline, standing in for
  // the model. CallId brands the correlation id a provider would issue.
  void (async () => {
    const result = await ctx.tools.execute({
      callId: CallId('demo-1'),
      name: 'greet',
      arguments: { name: 'Cordis' },
      signal: new AbortController().signal,
    })
    console.log('tool replied:', JSON.stringify(result.content))
  })()
}
```

Cada patrón de aquí viene de los capítulos anteriores: `inject: ['tools']` ([capítulo 3](03-services.es.md)) mantiene el plugin a la espera hasta que existe el registro de herramientas; `ctx.tools.register(...)` une el disposer del registro al plugin ([capítulo 2](02-lifecycle-and-effects.es.md)), así que descargarlo anula el registro de la herramienta. `defineTool` convierte la especificación de `parameters` en el JSON Schema que se le muestra al modelo, infiere el tipo de `args` y valida los argumentos proporcionados por el modelo antes de que se ejecute `execute`. La herramienta devuelve el valor canónico declarado por `output.schema`; `output.render` produce por separado el contenido de resultado Native y durable.

## Un plugin observador

Crea `tool-logger.ts` —un plugin separado que observa cada llamada de herramienta de la aplicación a través del evento `tools/result` del harness:

```ts
import type { Context } from '@deepseek-ai/cordis'
import type {} from '@deepseek-ai/dsh-tools'

export const name = 'tool-logger'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.on('tools/result', (exec, result) => {
    const text = result.content
      .map(block => (block.type === 'text' ? block.text : ''))
      .join('')
    console.log(`[tool-logger] ${exec.name} -> ${text}`)
  })
}
```

La línea `import type {} from '@deepseek-ai/dsh-tools'` incorpora los declaration merges del paquete para que `'tools/result'` y su payload estén tipados: el mismo movimiento que el import de `stats.ts` del capítulo 4, pero a escala de paquete.

## Compón y ejecuta

```yaml
- name: '@deepseek-ai/dsh-system-prompt'
- name: '@deepseek-ai/dsh-tools'
- name: './tool-logger.ts'
- name: './greet-tool.ts'
```

`@deepseek-ai/dsh-tools` hace inject del servicio `systemPrompt` porque las herramientas contribuyen schemas al system prompt, así que la composición lista también su provider. Sin él, el plugin de herramientas se queda en PENDING como se describe en el [capítulo 6](06-composition-and-hmr.es.md).

```sh
node --import tsx ../../vendor/cordis/bin.js
```

```
[tool-logger] greet -> Hello, Cordis!
tool replied: [{"type":"text","text":"Hello, Cordis!"}]
```

El logger se disparó primero: `tools/result` se emite como parte de la materialización del resultado, antes de que la promesa de `execute` se resuelva hacia quien la llamó. Ninguno de tus plugins sabe que el otro existe: el servicio de registro y el evento los conectan.

## De aquí a un agent (agente) completo

Un agent real es esta composición más plugins adicionales: un adaptador de LLM (modelo de lenguaje de gran tamaño), el agent loop, persistencia y un punto de entrada. Compara con [examples/headless-agent/cordis.yml](../../examples/headless-agent/cordis.yml): ya puedes leer cada entrada de ese archivo. Añade tu `greet-tool.ts` a una copia de ese archivo.

Dónde seguir:

- [Construye una herramienta](../user/develop/basic/tool.es.md) — más sobre `defineTool`, incluida la presentación y schemas más ricos.
- [Diseño de capacidades en tres capas](../user/develop/practice/index.es.md) — cómo estructura el harness las capacidades reemplazables.
- Las regiones generadas `cordis-surface` de las [páginas de subsistemas](../subsystems/core.es.md) — todo lo que puedes inject y a lo que puedes escuchar, cada cosa en su página correspondiente.
- [Arquitectura](../architecture.es.md) — el mapa del sistema en el que viven estos plugins.

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)
