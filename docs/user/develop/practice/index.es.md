# Diseño de capacidad de tres roles

[English](index.md) | Español

Esta página tiene dos partes: una referencia conceptual del patrón de capacidad de tres roles, seguida de un tutorial avanzado que construye una capacidad. Completa primero la [ruta básica de plugins](../basic/index.es.md) y el [tutorial de servicios](../framework/service.es.md).

## Referencia conceptual

Cuando una capacidad es lo bastante general como para necesitar providers reemplazables, como la ejecución de Bash, Harness separa tres roles: una **Service Definition**, un **Service Provider** y un **Consumer**. Pon los roles en paquetes separados cuando necesiten evolucionar o ser reemplazados de forma independiente; de lo contrario, un paquete puede ser dueño de más de un rol. La capacidad completa es su seam. Ningún rol individual es un seam.

## Ejemplo con Bash

La capacidad de ejecución de Bash se compone de:

- **Service Definition** (`dsh-shell`) — define el servicio de Cordis y los tipos de petición y resultado de Bash
- **Service Provider** (`dsh-bash-local`) — ejecuta comandos en la máquina local
- **Consumer** (`dsh-tool-bash`) — expone la capacidad como una herramienta que el modelo puede llamar

```
┌─────────────┐     ┌──────────────────┐     ┌──────────────┐
│  dsh-shell   │────▶│  dsh-bash-local  │     │ dsh-tool-bash│
│(definition) │     │    (provider)     │     │(consumer/tool)│
└─────────────┘     └──────────────────┘     └──────────────┘
       ▲                                            │
       └────────────────────────────────────────────┘
                    inject: ['shell']
```

## Beneficios de la división

### Reemplaza providers

Una Service Definition puede tener varios providers seleccionados a través de `cordis.yml`:

```yaml
# Local execution
- name: '@deepseek-ai/dsh-bash-local'

# Replace this row with another package that provides the same service.
```

La Service Definition y la herramienta permanecen sin cambios mientras el provider cambia.

### Evolucionan de forma independiente

- La Service Definition cambia rara vez después de que los llamadores dependan de su contrato.
- Los Service Providers pueden mejorar el rendimiento y la seguridad de forma independiente.
- Los Consumers pueden cambiar cómo presentan la capacidad al modelo.

### Desacopla dependencias

- El Service Provider depende de la Service Definition.
- El Consumer depende de la Service Definition.
- El Service Provider y el Consumer **no dependen el uno del otro**.

La [referencia de capability seams](../../../capability-seams.es.md) es dueña de las familias integradas actuales y de los enlaces a paquetes.

## Tutorial: desarrolla una capacidad de tres roles

### Paso 1: escribe la Service Definition

```ts ignore-check
// packages/my-cap/my-cap/src/index.ts
import { Service, type Context } from '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Context {
    myCap: MyCapService
  }
}

export abstract class MyCapService extends Service {
  constructor(ctx: Context) {
    super(ctx, 'myCap')
  }

  /** Execute the capability. */
  abstract execute(request: MyCapRequest): Promise<MyCapResult>
}

export interface MyCapRequest {
  input: string
}

export interface MyCapResult {
  output: string
}
```

### Paso 2: escribe un Service Provider

```ts ignore-check
// packages/my-cap/my-cap-local/src/index.ts
import type { Context } from '@deepseek-ai/cordis'
import { MyCapService, type MyCapRequest, type MyCapResult } from '@deepseek-ai/dsh-my-cap'

class MyCapLocal extends MyCapService {
  async execute(request: MyCapRequest): Promise<MyCapResult> {
    // Local provider behavior.
    return { output: request.input.toUpperCase() }
  }
}

export const name = 'my-cap-local'

export function apply(ctx: Context) {
  ctx.plugin(MyCapLocal)
}
```

### Paso 3: escribe un consumer

```ts ignore-check
// packages/my-cap/tool-my-cap/src/index.ts
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

export const name = 'tool-my-cap'
export const inject = ['tools', 'myCap']

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'my_cap',
    description: 'Execute my capability.',
    parameters: {
      input: { type: 'string', required: true },
    },
    output: {
      schema: { type: 'string' },
      render: (_args, value) => [{ type: 'text', text: value }],
    },
    async execute(args) {
      const result = await ctx.myCap.execute({ input: args.input })
      return result.output
    },
  }))
}
```

### Compónlos en cordis.yml

```yaml
- name: '@deepseek-ai/dsh-my-cap-local'
- name: '@deepseek-ai/dsh-tool-my-cap'
```

## Puntos de diseño

- **No dividas de forma preventiva** — usa paquetes separados solo cuando los roles necesiten evolucionar de forma independiente. Un plugin de herramienta simple no lo necesita.
- **La Service Definition es dueña de los tipos Request/Result** — los Service Providers y los Consumers dependen solo del paquete de la Service Definition.
- **Explícito > implícito** — resuelve los valores por defecto en un paso explícito `resolve(request): Spec` en lugar de ocultar expresiones `?? default` dentro de `run()`.

## Pasos siguientes

- [Adaptador de LLM](./llm-adapter.es.md) — implementa un provider de LLM
