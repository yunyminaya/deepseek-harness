# Recetario: añadir un adaptador de LLM

[English](adding-an-llm-adapter.md) | Español

Cómo conectar un nuevo provider de modelos. Implementaciones de referencia: `packages/llm/llm-deepseek` (HTTP directo, SSE (Server-Sent Events) delimitado por `eventsource-parser`) y `packages/llm/llm-pi-ai` (que envuelve una librería de LLM). Lee primero la documentación de `StreamChunk` en `packages/llm/llm/src/types.ts`: registra las convenciones de protocolo contra las que se verificaron ambos adaptadores.

## La forma

```ts ignore-check
class MyAdapter extends LlmAdapter {
  async * stream(options: GenerateOptions): AsyncIterable<StreamChunk> { … }
}

export const name = 'llm-myprovider'
export const inject = ['llm']
export const Config: z<Config> = z.object({ apiKey: z.string(), … })

export function apply(ctx: Context, config: Config) {
  ctx.llm.registerAdapter(['my-provider'], new MyAdapter(…))
}
```

El registro se basa en efectos (compatible con HMR (hot module replacement)); un adaptador por ruta de provider: los duplicados lanzan una excepción y el registro multirruta es de todo o nada. `options.provider` selecciona el adaptador y `options.model` es el id del modelo del provider, de modo que un adaptador con catálogo dinámico puede servir modelos nuevos sin reconfigurar el ciclo de vida. Los secretos son nativos de Cordis: Config de schemastery con respaldos de entorno (env), alimentada desde cordis.yml mediante `!!js process.env.MY_KEY`. Nunca leas archivos de claves ad hoc en el código.

## Obligaciones de protocolo (el contrato que verificaron dos implementaciones)

- Emite `usage` ANTES de `finish`; no emitas NADA después de `finish`. La forma robusta: almacena finish/usage en búfer hasta el marcador de fin de flujo del provider y luego vacíalo (así se manejan los providers que envían chunks finales de solo usage).
- Los `arguments` de una llamada de herramienta son cadenas JSON EN BRUTO de extremo a extremo; transmite los fragmentos como `argumentsDelta`. Si tu provider devuelve objetos ya analizados, vuelve a serializarlos como cadena en `block-end`.
- Asigna los `index` de los bloques en el orden en que aparecen por primera vez en el flujo; reutiliza el índice para cada delta del mismo bloque.
- Los errores tienen exactamente dos vías permitidas: LANZAR desde `stream()` (fallos de transporte y de protocolo — usa `LlmError` con un código estable), o terminar el flujo con `finish {kind: 'error' | 'aborted'}` (fallos dentro de la banda del provider). Los Consumers gestionan ambas; elige según la clase de fallo y documéntalo.
- Respeta `options.signal` (pásalo a fetch / a tu SDK).
- Un campo de `GenerateOptions` que tu provider no pueda respetar (p. ej. una lista `stop` en un provider sin secuencias de parada): lanza `LlmError(..., 'UNSUPPORTED')` en lugar de ignorarlo en silencio.
- Si el provider exige ids de respuesta, firmas u otros metadatos nativos en llamadas posteriores, emite la proyección JSON mínima sin pérdida como `finish.replayState`. Valídala al reconstruir el historial. `LlmRuntime` solo la pasa cuando la ruta histórica del provider y la ruta destino del provider están actualmente gestionadas por la misma instancia exacta del adaptador; tu adaptador decide si la restauración entre modelos iguales, entre modelos distintos o entre providers distintos es lícita. Nunca deduzcas la reproducción nativa solo a partir de los nombres del provider o del modelo cuando no hay estado.

Los interruptores del modo de pensamiento (thinking) específicos del provider permanecen en la Config del adaptador. Los metadatos exactos del modelo usan un único seam de capacidad neutral respecto al provider: implementa `resolveModel()` con la identidad de provider y modelo y los campos opcionales `context` y `reasoning`, declara un `defaultEffort` configurado solo cuando exista y respeta el `AbortSignal` opcional del resolver. Los esfuerzos de razonamiento son ids opacos ordenados que el adaptador asigna a las peticiones del provider. Conserva la lista seleccionable autoritativa del adaptador, incluido un `off` definido por el adaptador cuando se admita, sin exponer las grafías definitivas del wire ni forzar los valores no admitidos; un id no tiene por qué coincidir con su representación en el wire.

## Estructura de implementación

Mantén los tipos del wire, la serialización de peticiones, el análisis del transporte, la traducción de chunks y la clase del adaptador como responsabilidades separadas; [`llm-deepseek`](../../packages/llm/llm-deepseek/README.es.md) es el diseño de referencia.

## Verificación

Sigue la [política de pruebas del repositorio](../testing.es.md), que es la dueña de la cobertura de adaptadores, las comprobaciones con providers reales y los requisitos de las entradas publicadas.
