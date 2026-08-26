# Adaptadores de LLM

[English](llm-adapter.md) | Español

Esta guía conecta un nuevo provider de LLM (modelo de lenguaje de gran tamaño) con Harness.

## Panorama general

Un adaptador de LLM extiende `LlmAdapter` e implementa `stream()`, traduciendo la petición neutra respecto al provider de Harness en una llamada a la API del provider y traduciendo la respuesta de vuelta a chunks de Harness.

## Implementación mínima

```ts
import type { Context } from '@deepseek-ai/cordis'
import Schema from '@deepseek-ai/schemastery'
import { LlmAdapter, type GenerateOptions, type StreamChunk } from '@deepseek-ai/dsh-llm'

class MyAdapter extends LlmAdapter {
  private apiKey: string

  constructor(apiKey: string) {
    super()
    this.apiKey = apiKey
  }

  async *stream(options: GenerateOptions): AsyncIterable<StreamChunk> {
    // 1. Convert options.messages to the provider format.
    // 2. Call the streaming API.
    // 3. Convert the response into StreamChunk values.
  }
}

export interface Config {
  apiKey: string
  providers: string[]
}

export const Config: Schema<Config> = Schema.object({
  apiKey: Schema.string().required(),
  providers: Schema.array(Schema.string()).required(),
})

export const name = 'my-llm-adapter'
export const inject = ['llm']

export function apply(ctx: Context, config: Config) {
  const adapter = new MyAdapter(config.apiKey)
  ctx.llm.registerAdapter(config.providers, adapter)
}
```

## Protocolo de StreamChunk

`stream()` emite chunks usando este protocolo:

```ts
import { CallId, type StreamChunk } from '@deepseek-ai/dsh-llm'

async function* exampleChunks(): AsyncIterable<StreamChunk> {
  // 1. Start each content block with block-start.
  yield { type: 'block-start', index: 0, blockType: 'text' }

  // 2. Stream text through text-delta.
  yield { type: 'text-delta', index: 0, text: 'Hello' }
  yield { type: 'text-delta', index: 0, text: ' world' }

  // 3. End each content block with block-end and the complete block.
  yield {
    type: 'block-end',
    index: 0,
    block: { type: 'text', text: 'Hello world' },
  }

  // 4. Tool-call block.
  yield { type: 'block-start', index: 1, blockType: 'tool-call' }
  yield {
    type: 'tool-call-delta',
    index: 1,
    id: CallId('call-123'),
    name: 'bash',
    argumentsDelta: '{"command":"ls"}',
  }
  yield {
    type: 'block-end',
    index: 1,
    block: {
      type: 'tool-call',
      id: CallId('call-123'),
      name: 'bash',
      arguments: '{"command":"ls"}',
    },
  }

  // 5. Token usage.
  yield { type: 'usage', usage: { inputTokens: 100, outputTokens: 50 } }

  // 6. Finish reason.
  yield { type: 'finish', reason: { kind: 'stop' } }
  // Alternatively, { kind: 'tool-calls' } requests tool execution.
}
```

### Reglas clave

- Todo `block-start` tiene su `block-end` correspondiente.
- `index` aumenta desde 0 e identifica el orden de los bloques de contenido.
- Un `tool-call-delta` transporta texto JSON crudo en `argumentsDelta`, todo de una vez o repartido en varios chunks.
- `finish` es el chunk final.
- Emite `usage` antes de `finish`.

## GenerateOptions

`stream()` recibe el tipo exportado `GenerateOptions`. Incluye el modelo, el id de esfuerzo de razonamiento propiedad del adaptador, el historial de la conversación, el prompt del sistema, los schemas de herramientas, los parámetros de generación, las secuencias de parada y la señal de aborto; trata el tipo de TypeScript exportado por `@deepseek-ai/dsh-llm` como autoritativo. Mapea los campos admitidos a la API del provider. Si el provider no puede honrar un campo, lanza `LlmError` con un código estable en lugar de descartarlo silenciosamente.

Sobrescribe `resolveModel(provider, model, signal?)` para devolver la identidad exacta de provider/modelo más los metadatos opcionales `context` y `reasoning` en una sola consulta. Los metadatos de razonamiento contienen ids opacos ordenados y nombres mostrables, además de un valor por defecto configurado opcional; preserva la lista seleccionable autoritativa del adaptador, incluido `off` cuando la API de capacidad ascendente lo devuelve, en lugar de promover esos valores a un enum del core. Honra la señal opcional para la consulta asíncrona, de modo que la cancelación y la liberación alcancen la quietud. El servicio valida el agregado y rechaza los esfuerzos explícitos no admitidos antes de `stream()`; omitir `reasoning` significa que ese modelo no tiene capacidad de esfuerzo de razonamiento seleccionable.

## Registra un adaptador

```ts ignore-check
ctx.llm.registerAdapter(['my-provider'], adapter)
```

El primer argumento lista las rutas de provider que maneja el adaptador. `GenerateOptions.provider` selecciona el adaptador registrado, mientras que `GenerateOptions.model` pasa un id de modelo propiedad del adaptador sin registro de ciclo de vida. Sobrescribe `listModels()` cuando el adaptador pueda anunciar opciones de modelo a los selectores.

## Úsalo desde cordis.yml

```yaml
- id: my-llm
  name: './src/my-llm-adapter.ts'
  config:
    apiKey: !!js process.env.MY_API_KEY
    providers:
      - my-provider

- id: agent-loop
  name: '@deepseek-ai/dsh-agent-loop'
  config:
    agents:
      - id: main
        provider: my-provider
        model: my-model-v1
```

## Implementaciones de referencia

El repositorio contiene implementaciones completas:

- `packages/llm/llm-deepseek/` — adaptador de la API de DeepSeek que usa el formato compatible con OpenAI
- `packages/llm/llm-pi-ai/` — adaptador de Pi AI que usa un formato de API diferente

Compara los dos adaptadores incluidos para ver el mismo contrato del harness implementado sobre distintos SDKs de provider.

## Manejo de errores

Los adaptadores lanzan los fallos de transporte y de protocolo como valores `LlmError` con códigos estables. El agent loop conserva el error y el código para diagnóstico y política; no convierte un `Error` ordinario automáticamente. Cada petición HTTP al provider también debe fusionar `attributionHeaders()` y reenviar `options.signal`.

```ts
import {
  attributionHeaders,
  LlmAdapter,
  LlmError,
  type GenerateOptions,
  type StreamChunk,
} from '@deepseek-ai/dsh-llm'

class HttpAdapter extends LlmAdapter {
  constructor(private readonly endpoint: string) {
    super()
  }

  async *stream(options: GenerateOptions): AsyncIterable<StreamChunk> {
    const response = await fetch(this.endpoint, {
      method: 'POST',
      headers: {
        'content-type': 'application/json',
        ...attributionHeaders(),
      },
      body: JSON.stringify({ model: options.model, messages: options.messages }),
      ...options.signal ? { signal: options.signal } : {},
    })
    if (!response.ok) {
      throw new LlmError(`Provider API error: ${response.status}`, 'PROVIDER_HTTP_ERROR')
    }
    // A real adapter parses the response and emits the complete chunk sequence.
    yield { type: 'finish', reason: { kind: 'stop' } }
  }
}
```
