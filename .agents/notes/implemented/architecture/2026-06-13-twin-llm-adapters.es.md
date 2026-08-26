# Agent Note: Dos adaptadores LLM como gemelo de verificación de diseño

Status: implemented

[English](2026-06-13-twin-llm-adapters.md) | Español

## Problema

`dsh-llm` es dueño de un vocabulario de streaming neutro respecto al provider — el protocolo `StreamChunk` (`block-start`, `text-delta`, `reasoning-delta`, `tool-call-delta`, `block-end`, `usage`, `finish`) y los tipos de content blocks ([el vocabulario de content blocks](2026-06-11-content-block-vocabulary.es.md)). Un vocabulario definido contra un único adaptador corre el riesgo de incrustar las peculiaridades de ese adaptador en el contrato «neutro»: cualquier cosa que la única implementación haga se convierte en la especificación de facto, y la abstracción queda sin verificar hasta que llega un segundo provider — momento en el que la fuga es cara de arreglar.

## Decisión

Publica **dos** adaptadores contra el mismo contrato desde el principio, construidos deliberadamente sobre internals distintos:

- `dsh-llm-deepseek` — `fetch` directo + traducción dentro del repo contra la API de DeepSeek; el enmarcado SSE se delega en `eventsource-parser` ([el cambio de parser SSE archivado](../../archived/simplification/2026-07-26-eventsource-parser-for-deepseek-sse.md)). La identidad gemela consiste en ser dueño de los internals de fetch/traducción en lugar de delegar en un SDK de provider completo, no en implementar a mano la fontanería del transporte.
- `dsh-llm-pi-ai` — el mismo endpoint a través de la librería `@earendil-works/pi-ai` (con su propio vocabulario de eventos).

La regla que imponen: **cualquier cosa que el vocabulario de StreamChunk no pueda expresar para AMBAS implementaciones es un bug del vocabulario central**, detectado de inmediato y no cuando llegue el siguiente provider. El par fijó convenciones hoy documentadas en `StreamChunk` en `dsh-llm/src/types.ts`: usage emitido antes de finish, nada después de finish, los `arguments` de tool-call como strings JSON crudos de extremo a extremo, y las dos rutas de error sancionadas (lanzar desde `stream()` *o* terminar con `finish {kind:'error'|'aborted'}`) que un consumidor debe manejar en ambos lados — una divergencia que el adaptador respaldado por la librería sacó a la luz y que un único adaptador de fetch directo habría ocultado.

## Alternativas consideradas

- **Un único adaptador** — menos código y la mitad del coste e2e, pero deja sin verificar la afirmación de «neutro respecto al provider»; el vocabulario codificaría en silencio los supuestos de DeepSeek-vía-fetch.
- **Un segundo adaptador mock** — más barato, pero no ejercita las peculiaridades de cable de un provider real, así que demuestra poco. El gemelo es real-contra-real.

## Consecuencias

El gemelo duplica el mantenimiento del adaptador y del e2e condicionado a clave —ambos cubren V4 Flash y Pro en modos de razonamiento representativos— a cambio de una validación continua de la neutralidad del seam y de un segundo ejemplo de implementación. Ambos usan `apiKey`, `baseURL` y `models`; el adaptador de fetch directo expone `thinking`/`reasoningEffort`, mientras que pi-ai expone un único nivel `reasoning`. Una futura suite de conformidad podría justificar retirar uno de los adaptadores mediante un Agent Note que lo sustituya.
