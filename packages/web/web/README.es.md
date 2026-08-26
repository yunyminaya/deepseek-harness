# @deepseek-ai/dsh-web

[English](README.md) | [中文](README.zh.md) | Español

El **`WebRuntime`** (`ctx.web`) define QUÉ acceso web tiene el harness — buscar en la web, obtener una URL — sobre múltiples providers, sin atar el contrato del modelo a la forma de API de un único proveedor.

Este paquete es dueño del rol de Service Definition de la capacidad web. A diferencia de shell/fs abarca dos operaciones (búsqueda y obtención) en un solo seam, con potencialmente múltiples providers en cada una:

| Paquete | Rol |
|---|---|
| `@deepseek-ai/dsh-web` (este) | Service Definition: el servicio, los registros de providers, la política de selección, el vocabulario de peticiones/resultados y la taxonomía `WebError` |
| `@deepseek-ai/dsh-web-search-exa` | Provider de búsqueda: Exa |
| `@deepseek-ai/dsh-web-search-perplexity` | Provider de búsqueda: Perplexity |
| `@deepseek-ai/dsh-web-fetch-http` | Provider de obtención: HTTP(S) público anónimo |
| `@deepseek-ai/dsh-tool-web` | Consumer: los schemas de herramientas `web_search` / `web_fetch` orientadas al modelo sobre `ctx.web` |

La búsqueda y la obtención no comparten schema de petición ni lógica de negocio, pero son deliberadamente un solo seam: `ctx.web` es una única capa intermedia de acceso web con un único dueño de la política de selección de providers, un único vocabulario de aborto/error y una única superficie de configuración de cara al producto («cómo llega este harness a la web»). Los pares de métodos `Search`/`Fetch` son deliberadamente paralelos.

## API de servicio (`ctx.web`)

| Miembro | Semántica |
|---|---|
| `registerSearchProvider(provider)` / `registerFetchProvider(provider)` | Registra un backend. Lanza `WebError` `WEB_DUPLICATE_PROVIDER` ante un id duplicado dentro de esa clase de capacidad. Devuelve un disposer. Se destruye con la fibra que lo llama. |
| `search(request, signal?)` | Resuelve el provider de búsqueda y ejecuta una búsqueda. Aplica `request.maxResults` sobre el resultado (trunca `sources[]`, fija `truncated`). Lanza `WebError` cuando la capacidad no puede ejecutarse. |
| `fetch(request, signal?)` | Resuelve el provider de obtención y recupera una URL. Una respuesta no-2xx es un resultado, no un lanzamiento. Lanza `WebError` ante fallos para recuperar o representar el recurso con seguridad. |

Los providers registran **capacidades**, no herramientas. `dsh-tool-web` es el único dueño de los nombres, descripciones, guías de prompt, JSON schemas y presentación orientados al modelo.

## Selección

La selección nunca depende del orden de registro, de la configuración ni del HMR. Una capacidad tiene un id de provider explícito (config `searchProvider`/`fetchProvider`, o env `$DSH_WEB_SEARCH_PROVIDER`/`$DSH_WEB_FETCH_PROVIDER` que alimenta los mismos campos), o autoselecciona cuando hay exactamente un provider utilizable registrado. `search()`/`fetch()` resuelven el provider en tiempo de ejecución:

| Situación | Ejecución |
|---|---|
| id configurado, registrado y `available()` | ejecuta ese provider |
| id configurado no registrado | `WEB_PROVIDER_CONFIGURED_MISSING` |
| id configurado, registrado pero no disponible | `WEB_PROVIDER_CONFIGURED_UNAVAILABLE` |
| sin id, exactamente un provider utilizable registrado | lo ejecuta |
| sin id, ningún provider utilizable | `WEB_PROVIDER_UNAVAILABLE` |
| sin id, múltiples providers utilizables | `WEB_PROVIDER_AMBIGUOUS` |

Las ramas de fallo lanzan `WebError`, cuyo código estructurado (más el detalle del mensaje — el id ausente, el conjunto de candidatos ambiguos) es el que enrutan los llamadores directos. El `available()` de un provider es una comprobación local barata (presencia de credenciales, configuración analizable) que alimenta esta selección en tiempo de ejecución y **no debe hacer llamadas de red**; `dsh-tool-web` nunca lo llama — la herramienta ejecuta a través de `ctx.web.search()`/`fetch()` y enruta según los códigos lanzados, de modo que la selección de providers tiene un único dueño.

## Vocabulario

`WebSearchRequest` (`query`, `maxResults?`) → `WebSearchResult` (`content?`, `sources[]`, `truncated`); cada `WebSearchSource` lleva una `url` obligatoria y `title`/`snippet`/`publishedAt` opcionales (las citas de Perplexity pueden ser de solo URL). `WebFetchRequest` (`url`) → `WebFetchResult` (`url` final, `statusCode`, `body`, `truncated`); la cancelación es un argumento `AbortSignal` opcional directo a `search()`/`fetch()`. `WebFetchBody` es una unión discriminada CERRADA (`html` | `text`) propia de este paquete — los consumers hacen `switch` a la exhaustividad, así que un nuevo tipo rompe su compilación hasta que se gestione. Ver `src/types.ts` para los contratos completos y la taxonomía de códigos `WebError`.

## Experiencia del modelo

Indirectamente, a través de `dsh-tool-web`, que retiene datos normalizados acotados del provider o los fallos exactos de provider-configurado, provider-no-disponible, sin-provider, múltiples-providers y `Error: <message>` mientras este registro no aporta prompt ni schema por sí mismo.

#### Efecto de KV Cache

Sin invalidación directa; el consumer nombrado es dueño de cualquier cambio de prefijo de petición.

## Limitaciones conocidas y trabajo diferido

- **No hay superficie de observación** — no hay evento de cambio de provider ni consulta de estado de la capacidad; la disponibilidad se observa únicamente ejecutando `search()`/`fetch()` y enrutar los códigos `WebError` lanzados, y el fallo sin-provider es el genérico `WEB_PROVIDER_UNAVAILABLE` sin enumeración de razones por provider ([Agent Note](../../../.agents/notes/archived/simplification/2026-07-04-drop-unconsumed-web-observation-surface.md)).
- **`WebSearchRequest` lleva solo `query` + `maxResults`** — los controles neutrales para el provider (antigüedad, filtros de dominio, indicios regionales, profundidad de búsqueda) quedan diferidos hasta que Exa y Perplexity puedan honrarlos ambos con honestidad ([Agent Note del seam](../../../.agents/notes/implemented/architecture/2026-06-24-web-capability-seam.es.md)).
- **`WebFetchBody` no tiene rama `pdf`** — el soporte de PDF con texto extraíble es trabajo diferido con nombre; la unión cerrada hace que añadirlo sea un cambio impuesto por el compilador en los tres paquetes web.
- **La extracción de páginas respaldada por providers queda fuera del alcance de `fetch()`** — una capacidad `web_extract` al estilo Firecrawl/Tavily queda diferida en lugar de ampliar la operación de obtención.
