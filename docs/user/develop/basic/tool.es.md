# Crea una herramienta

[English](tool.md) | Español

Este tutorial añade una herramienta `greet` a la Web UI. Completa primero [Tu primer plugin](./index.es.md) y conserva su directorio `scratch-plugin`.

## Crea el plugin de la herramienta

Reemplaza `scratch-plugin/src/my-plugin.ts` por:

```ts
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

export const name = 'greet-tool'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'greet',
    description: 'Greet someone by name.',
    parameters: {
      name: { type: 'string', required: true, description: 'The name to greet' },
    },
    output: {
      schema: { type: 'string' },
      render: (_args, value) => [{ type: 'text', text: value }],
    },
    async execute(args) {
      return `Hello, ${args.name}!`
    },
  }))
}
```

`inject` hace que Cordis espere al registro de herramientas. `defineTool` infiere y valida `args` a partir de `parameters`; `execute` devuelve el valor canónico declarado por `output.schema`, y `output.render` convierte ese valor en contenido dirigido al modelo.

## Ejecuta y llama a la herramienta

Reinicia el comando de desarrollo si no está en ejecución:

```sh
pnpm dsh web --patch ./scratch-plugin/cordis.yml
```

Abre `http://127.0.0.1:3080` y pide: `Use the greet tool to greet Ada.` El modelo puede llamar a `greet` y recibe `Hello, Ada!` como resultado de la herramienta.

## Pasos siguientes

- [Configuración de plugin](./config.es.md) — haz configurable el saludo.
- [Referencia de creación de herramientas](../../../cookbook/adding-a-tool.es.md) — consulta schemas anidados, valores canónicos, trabajo en segundo plano, hooks de política, Code Mode y tarjetas de UI.
- [Capas de capacidad](../practice/index.es.md) — divide una capacidad reemplazable en paquetes de Service Definition, Service Provider y Consumer.
