# @deepseek-ai/dsh-web-search-perplexity

[English](README.md) | [中文](README.zh.md) | Español

Un `WebSearchProvider` respaldado por [Perplexity](https://perplexity.ai) para el [seam de capacidad web](../web/README.es.md) del harness (`ctx.web`). Llama al endpoint `POST /chat/completions` compatible con OpenAI de Perplexity y mapea la respuesta generada más las citas al `WebSearchResult` normalizado del seam.

Este es un paquete de **implementación**: registra un provider en `ctx.web`, no es dueño de la clave y no registra una herramienta orientada al modelo. Como `@deepseek-ai/dsh-llm-deepseek`, es un plugin de función/espacio de nombres (`inject: ['web']`). La forma del cable compatible con OpenAI es un detalle privado del provider — no hace que este provider dependa de `ctx.llm`.

## Configuración

| Clave | Predeterminado | Significado |
|---|---|---|
| `apiKey` | `$PERPLEXITY_API_KEY` | Clave de API de Perplexity. Si está vacía o ausente, el provider no está disponible. |
| `baseURL` | `https://api.perplexity.ai` | Base del endpoint; se añade `/chat/completions`. Un valor no analizable hace que el provider no esté disponible. |
| `model` | `sonar` | Nombre del modelo de búsqueda. |
| `maxTokens` | `1024` | Tope superior de tokens de la respuesta generada (`max_tokens`). Debe ser un entero positivo. |
| `searchRecency` | (sin definir) | Ventana de antigüedad enviada como `search_recency_filter`: `day`, `week`, `month` o `year`. Sin definir, no se envía filtro. |

```yaml
- id: web-search-perplexity
  name: '@deepseek-ai/dsh-web-search-perplexity'
  config:
    apiKey: *** process.env.PERPLEXITY_API_KEY
```

## Mapeo

`content` ← `choices[0].message.content` (la respuesta generada). `sources[]` prefiere los `search_results[]` estructurados (`url`, `title`, `snippet`, `publishedAt` ← `date`), con el array `citations[]` de solo URLs como alternativa únicamente cuando `search_results` está ausente — esas fuentes llevan solo una `url`, razón por la que `title`/`snippet`/`publishedAt` son opcionales en el seam. Los fallos del provider surgen como `WebError` `WEB_PROVIDER_ERROR`; una petición abortada surge como `WEB_ABORTED`. Las redirecciones HTTP se rechazan antes de contactar con el destino de `Location` y surgen como `WEB_PROVIDER_ERROR`. Perplexity no tiene control del número de resultados, así que `maxResults` lo aplica el seam (truncando `sources[]` y fijando `truncated`).

## Experiencia del modelo

### Petición auxiliar a Perplexity

#### Lo que ve el modelo

Un modelo de Perplexity aparte recibe `<query>` literal como su único mensaje de usuario a través del endpoint de chat-completions. Esta petición no forma parte del contexto del modelo de la conversación.

#### Efecto de tokens

Se incurre en tokens propios del provider por cada búsqueda; `maxTokens` limita la respuesta generada.

#### Efecto de KV Cache

Independiente de la caché de la petición de la conversación. Una consulta idéntica bajo la misma ruta de modelo puede reutilizar la caché del provider; una consulta o una ruta distintas establecen un prefijo diferente.

### Resultado de la herramienta en la conversación, indirectamente

#### Lo que ve el modelo

A través de [`dsh-tool-web`](../tool-web/README.es.md), el modelo de la conversación ve la respuesta generada más metadatos estructurados del resultado o citas de solo URL. Los fallos exactos de este provider son `Perplexity search aborted`, `Perplexity search request failed: <error>` y `Perplexity returned an unprocessable response body: <error>`; los fallos HTTP conservan el mensaje del provider. El consumer es dueño del envoltorio del error.

#### Efecto de tokens

Cero tokens directos de la conversación por el registro. Los tokens de la respuesta y de las fuentes dependen de los datos, el recuento de fuentes está acotado por el servicio y el resultado o error retenido se reenvía hasta la compactación.

#### Efecto de KV Cache

Solo append; el contenido recién visible sigue al prefijo reutilizable de la petición y no invalida las entradas existentes de la caché KV.

## Limitaciones conocidas y trabajo diferido

- **Las fuentes del fallback de citas son de solo URL** — cuando Perplexity omite los `search_results[]` estructurados, las fuentes no llevan `title`/`snippet`/`publishedAt`, así que la herramienta renderiza etiquetas de solo nombre de host.
- **Las fuentes devueltas en exceso siguen costando tokens y latencia** — sin control del número de resultados en el cable, `maxResults` se aplica solo a posteriori mediante el truncamiento del seam.
- **Solo se exponen `model`/`maxTokens`/`searchRecency`** — los demás controles de búsqueda de Perplexity (filtros de dominio, tamaño de contexto de `web_search_options`, imágenes) esperan campos neutrales para el provider en la Service Definition ([Agent Note del seam](../../../.agents/notes/implemented/architecture/2026-06-24-web-capability-seam.es.md)).
- **La clasificación del aborto se basa en la forma del error** — solo una `DOMException` llamada `AbortError` se mapea a `WEB_ABORTED`; un aborto que lleva una razón personalizada (p. ej. el `TimeoutReason` de `dsh-timeout`) surge como `WEB_PROVIDER_ERROR`.
