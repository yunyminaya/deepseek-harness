# @deepseek-ai/dsh-web-search-deepseek

[English](README.md) | Español

Un `WebSearchProvider` respaldado por [DeepSeek](https://deepseek.com) para el [seam de capacidad web](../web/README.es.md) del harness (`ctx.web`). Llama a la **API Messages compatible con Anthropic** de DeepSeek (`POST {baseURL}/messages`) con la herramienta de servidor nativa `web_search_20250305` habilitada, y mapea los bloques `web_search_tool_result` estructurados que DeepSeek devuelve al `WebSearchResult` normalizado del seam.

Este es un paquete de **implementación**: registra un provider en `ctx.web`, resuelve su credencial para cada búsqueda a través del seam opcional `ctx.credentials`, registra la petición auxiliar en la sesión del Agent iniciador cuando existe una, y no registra una herramienta orientada al modelo. Como `@deepseek-ai/dsh-llm-deepseek`, es un plugin de función/espacio de nombres (`inject: ['web']`). La forma del cable de Anthropic es un detalle privado del provider — no hace que este provider dependa de `ctx.llm`.

## En qué se diferencia de un endpoint de búsqueda dedicado

Exa y Perplexity exponen endpoints de búsqueda dedicados; DeepSeek no. En su lugar, este provider emite una **llamada completa a un modelo Messages** que lleva la herramienta de servidor `web_search`, así que una búsqueda cuesta un turno completo de modelo en latencia y tokens — más pesada que un endpoint de recuperación pura. DeepSeek ejecuta la búsqueda del lado del servidor y devuelve bloques `web_search_tool_result` **estructurados**; el provider analiza esos bloques y **nunca raspa URLs de la prosa del modelo**.

**Modo estricto**: si la respuesta no lleva ningún bloque `web_search_tool_result` (la búsqueda nativa no se disparó), el provider lanza `WebError` `WEB_PROVIDER_ERROR` en lugar de degradarse a raspar la prosa.

Reutiliza la referencia de credencial `DEEPSEEK_API_KEY` (ningún secreto nuevo) pero **no** `$DEEPSEEK_BASE_URL`: el endpoint de búsqueda es la base compatible con Anthropic (`https://api.deepseek.com/anthropic/v1`), distinta de la base de chat-completions (`https://api.deepseek.com`) que usa el adaptador LLM. Un servicio de credenciales montado es autoritativo; sin uno, el provider cae al entorno del proceso lanzador. La referencia se resuelve para cada búsqueda, así que una clave almacenada o rotada por la página Web Models alcanza la siguiente llamada sin reinicio.

## Configuración

| Clave | Predeterminado | Significado |
|---|---|---|
| `apiKey` | omitido | Clave de API de DeepSeek literal. Prefiere `apiKeyEnv` para que ningún secreto entre en la configuración; un literal no vacío prevalece. |
| `apiKeyEnv` | `DEEPSEEK_API_KEY` | Referencia de credencial resuelta para cada búsqueda a través de `ctx.credentials`, o desde el entorno del proceso cuando ese seam está ausente. Un valor ausente hace fallar la llamada como `WEB_PROVIDER_CREDENTIAL_MISSING`. |
| `baseURL` | `https://api.deepseek.com/anthropic/v1` | Base del endpoint compatible con Anthropic; se añade `/messages`. Recae a `$DEEPSEEK_SEARCH_BASE_URL` desde cualquier capa de entorno; no reutilices `$DEEPSEEK_BASE_URL`, que pertenece al adaptador LLM de chat-completions. Un valor no analizable hace que el provider no esté disponible. |
| `model` | `deepseek-v4-flash` | Nombre del modelo en formato Anthropic. |
| `apiVersion` | `2023-06-01` | Valor de la cabecera `anthropic-version`. |
| `maxTokens` | `4096` | Tope superior de entero positivo sobre los tokens generados para la petición Messages. |
| `maxUses` | `5` | Máximo de entero positivo de usos de la herramienta de servidor `web_search` por petición. |

```yaml
- id: web-search-deepseek
  name: '@deepseek-ai/dsh-web-search-deepseek'
  config:
    apiKeyEnv: DEEPSEEK_API_KEY
    baseURL: https://gateway.internal/anthropic/v1
```

La entrada anterior es la capa base de la sección Settings de `web-search-deepseek`: una capa de usuario sobre ella alcanza la búsqueda SIGUIENTE, porque el provider proyecta la sección por llamada en lugar de capturarla en el registro. La selección de provider del seam nunca parpadea, por tanto, cuando cambia un endpoint o un modelo. `apiKey` lleva `role('secret')`, así que nunca viaja en una respuesta `describe()` en ninguna capa — una superficie de configuración solo aprende si el dominio de credenciales guarda un valor para la referencia que `apiKeyEnv` nombra, nunca si una capa lleva una clave literal.

## Mapeo

DeepSeek no devuelve contenido de respuesta generado por el provider que este provider confíe como `content`, así que se omite `content`. `sources[]` viene de los ítems `web_search_result` dentro de los bloques `web_search_tool_result`: `url` ← `url`, `title` ← `title` y `publishedAt` ← `page_age`. Los snippets viven aparte como entradas `cited_text` claveadas por URL en las `citations[]` de un bloque de texto; el provider las une, dejando `snippet` ausente cuando no existe ningún extracto.

Los resultados se desduplican por URL porque una petición puede hacer aflorar la misma página a través de varias búsquedas. DeepSeek expone `maxUses`, no una perilla de recuento de resultados, así que el seam aplica `maxResults` truncando `sources[]` y fijando `truncated`.

Los fallos del provider se convierten en `WEB_PROVIDER_ERROR`; la cancelación del llamador se convierte en `WEB_ABORTED`. Las redirecciones HTTP se rechazan antes de contactar con el destino de `Location` y surgen como `WEB_PROVIDER_ERROR`.

## Registro de peticiones

Inmediatamente antes del despacho, una búsqueda que corre bajo un Agent iniciador añade el evento de sesión de solo registro `web/deepseek-search-llm-request`. Contiene el endpoint resuelto, la versión de API y el cuerpo JSON exacto sin secretos enviado a DeepSeek; las cabeceras y las credenciales quedan excluidas. Los fallos de credencial y las cancelaciones anteriores al despacho no crean ningún evento, mientras que los fallos HTTP o de respuesta posteriores dejan la petición intentada de forma duradera. Las llamadas programáticas directas al provider fuera de un Agent no tienen sesión iniciadora que registrar.

## Experiencia del modelo

### Petición auxiliar de búsqueda de DeepSeek

#### Lo que ve el modelo

Un modelo de DeepSeek aparte recibe exactamente `Perform a web search for the query: <query>` como su texto de usuario y una definición de la herramienta de servidor nativa `web_search`. Esta petición no forma parte del contexto del modelo de la conversación.

#### Efecto de tokens

Se incurre en tokens de entrada y salida propios del provider por cada búsqueda; `maxTokens` limita la salida generada y `maxUses` limita los usos de búsqueda nativa.

#### Efecto de KV Cache

Independiente de la caché de la petición de la conversación. La instrucción auxiliar y la definición de herramienta nativa pueden formar un prefijo estable, pero cada consulta cambiada o ruta de modelo distinta impide la reutilización desde su primera diferencia.

### Resultado de la herramienta en la conversación, indirectamente

#### Lo que ve el modelo

A través de [`dsh-tool-web`](../tool-web/README.es.md), el modelo de la conversación ve URLs desduplicadas, títulos, fechas y snippets de cita de los bloques estructurados de búsqueda; no se confía en la prosa del provider como respuesta. Los fallos exactos de este provider son el mensaje accionable de credencial ausente, `DeepSeek search credential resolution failed: <error>`, `DeepSeek search aborted`, `DeepSeek search request failed: <error>`, `DeepSeek returned no web_search_tool_result blocks; the request may not have triggered native web search` y `DeepSeek returned an unprocessable response body: <error>`; los fallos HTTP conservan el mensaje del provider. El consumer es dueño del envoltorio del error.

#### Efecto de tokens

Cero tokens directos de la conversación por el registro. Los tokens del resultado escalan con las fuentes y snippets devueltos, y luego el seam aplica el tope de fuentes solicitado.

#### Efecto de KV Cache

Solo append; el contenido recién visible sigue al prefijo reutilizable de la petición y no invalida las entradas existentes de la caché KV.

## Limitaciones conocidas y trabajo diferido

- **Una búsqueda cuesta un turno completo de modelo Messages** — latencia más tokens generados, con hasta `maxUses` búsquedas del lado del servidor; DeepSeek no expone ningún endpoint de recuperación dedicado.
- **La disponibilidad dinámica de credenciales se resuelve dentro de la operación** — el contrato síncrono `available()` puede establecer que existe un resolver, pero no puede consultar un almacén de credenciales asíncrono. Un provider seleccionado sin clave hace fallar la búsqueda con `WEB_PROVIDER_CREDENTIAL_MISSING`; el schema `web_search` estable permanece registrado. La cancelación del llamador compite con este preflight localmente, pero no puede forzar a un backend de credenciales arbitrario a detener su trabajo.
- **Las fuentes devueltas en exceso siguen costando tokens** — sin perilla de recuento de resultados en el cable, `maxResults` se aplica solo a posteriori mediante el truncamiento del seam.
- **Los resultados no citados no llevan `snippet`** — una fuente solo gana uno cuando una cita de bloque `text` (`cited_text`) coincide con su URL.
