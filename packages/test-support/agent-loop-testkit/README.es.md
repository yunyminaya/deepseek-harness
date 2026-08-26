# `@deepseek-ai/dsh-agent-loop-testkit`

[English](README.md) | Español

Montaje de prerrequisitos compartidos para tests que ejercitan el `AgentLoop` concreto. `mountAgentLoopTestDependencies(ctx, options?)` instala los servicios de LLM, sesión, system-prompt, herramienta y agent en orden de dependencia, y luego retorna antes de que el loop se monte.

Quien llama registra adaptadores y plugins opcionales, monta `AgentLoop` con la configuración bajo prueba y dispone su propio Context. La configuración del system-prompt y del registro de herramientas puede reenviarse a través de `options`; el helper no proporciona valores por defecto de prueba más allá de los que son propiedad de los servicios. Un fallo de carga de plugin rechaza la llamada al helper, mientras que los servicios activados antes en la secuencia siguen siendo propiedad del Context de quien llama.

```ts
import { Context } from '@deepseek-ai/cordis'
import AgentLoop from '@deepseek-ai/dsh-agent-loop'
import { mountAgentLoopTestDependencies } from '@deepseek-ai/dsh-agent-loop-testkit'

const ctx = new Context()

await mountAgentLoopTestDependencies(ctx)
// Register the test adapter and any optional plugins here.
await ctx.plugin(AgentLoop, { agents: [] })
```

Los tests de fallos de inyección, topología parcial, orden de carga de servicios o desmontaje de servicios montan sus dependencias directamente en lugar de usar este helper.

## Model Experience

Ninguna, ya que este helper de composición solo de tests ni impulsa ni modifica peticiones de modelo.

#### Efecto de caché KV

Ninguno; este paquete ni ensambla ni envía una petición de provider.

## Limitaciones conocidas y trabajo diferido

- **Solo se comparte la columna vertebral de prerrequisitos obligatorios** — los adaptadores, los plugins opcionales, `AgentLoop`, los agents y el desmontaje del Context siguen siendo propiedad de quien llama para que el orden específico del escenario siga visible.
