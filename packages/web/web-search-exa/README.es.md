# @deepseek-ai/dsh-web-search-exa

[English](README.md) | Español

Un `WebSearchProvider` respaldado por [Exa](https://exa.ai) para el [seam de capacidad web](../web/README.es.md) del harness (`ctx.web`). Llama al endpoint `POST /search` de Exa con contenidos resaltados y mapea el `results[]` plano al `WebSearchResult` normalizado del seam.

Este es un paquete de **implementación**: registra un provider en `ctx.web`, no es dueño de la clave `ctx.web` y no registra una herramienta orientada al modelo (eso es `@deepseek-ai/dsh-tool-web`). Como `@deepseek-ai/dsh-llm-deepseek`, es un plugin de función/espacio de nombres (`inject: ['web']`) que registra su backend, no un servicio con exportación por defecto.

## Configuración

| Clave | Predeterminado | Significado |
|---|---|---|
| `apiKey` | `$EXA_API_KEY` | Clave de API de Exa. Si está vacía o ausente, el provider no está disponible. |
| `baseURL` | `https://api.exa.ai` | Base del endpoint; se añade `/search`. Un valor no analizable hace que el provider no esté disponible. |
| `searchType` | `auto` | Modo de recuperación enviado como el `type` de Exa: `auto` (decide Exa), `keyword` o `neural`. |
| `numResults` | (sin definir) | Recuento de resultados por defecto cuando una petición no lleva `maxResults`. Sin definir, no se envía ningún valor por defecto. Debe ser un entero positivo. |
| `highlightsPerResult` | `1` | Frases resaltadas solicitadas por resultado (el `highlightsPerUrl` de Exa). Debe ser un entero positivo. |

```yaml
- id: web-search-exa
  name: '@deepseek-ai/dsh-web-search-exa'
  config:
    apiKey: *** process.env.EXA_API_KEY
```

## Mapeo

Exa devuelve un `results[]` plano y ninguna respuesta generada, así que se omite `content`. Cada resultado se mapea a un `WebSearchSource`: `url` ← `url`, `title` ← `title`, `snippet` ← la primera entrada no vacía de `highlights[]` (un resultado sin resaltado no tiene snippet portable y se descarta), `publishedAt` ← `publishedDate`. El `maxResults` de una petición prevalece sobre el valor por defecto `numResults` configurado y se envía como el `numResults` de Exa a modo de optimización de coste/latencia; el bound final lo aplica el seam. Los fallos del provider (errores HTTP, fallo de red, cuerpos no analizables o con forma incorrecta) surgen como `WebError` `WEB_PROVIDER_ERROR`; una petición abortada surge como `WEB_ABORTED`. Las redirecciones HTTP se rechazan antes de contactar con el destino de `Location` y surgen como `WEB_PROVIDER_ERROR`.

## Experiencia del modelo

Indirectamente, a través de [`dsh-tool-web`](../tool-web/README.es.md), que retiene las URLs acotadas por el `maxResults` de este provider, los títulos, los primeros resaltados y las fechas de publicación de este provider, o sus fallos exactos `Exa search aborted`, `Exa search request failed: <error>` y `Exa returned an unprocessable response body: <error>` bajo el envoltorio de error del consumer, mientras que las respuestas generadas y los campos privados del provider permanecen fuera del contexto.

#### Efecto de KV Cache

Sin invalidación directa; el consumer nombrado es dueño de cualquier cambio en el prefijo de petición.

## Limitaciones conocidas y trabajo diferido

- **Un resultado sin resaltado no vacío se descarta por completo** — no hay snippet portable que mapear, así que pueden devolverse menos fuentes que el recuento solicitado.
- **Solo se exponen `searchType`/`numResults`/`highlightsPerResult`** — los demás controles de Exa (livecrawl, categoría, filtros de dominio/fecha, contenidos de texto completo) esperan campos neutrales para el provider en la Service Definition ([Agent Note del seam](../../../.agents/notes/implemented/architecture/2026-06-24-web-capability-seam.es.md)).
- **La clasificación del aborto se basa en la forma del error** — solo una `DOMException` llamada `AbortError` se mapea a `WEB_ABORTED`; un aborto que lleva una razón personalizada (p. ej. el `TimeoutReason` de `dsh-timeout`) surge como `WEB_PROVIDER_ERROR`.
