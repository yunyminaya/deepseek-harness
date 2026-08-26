# @deepseek-ai/dsh-web-fetch-http

[English](README.md) | Español

Un `WebFetchProvider` HTTP(S) público y anónimo para el [seam de capacidad web](../web/README.es.md) del harness (`ctx.web`). Recupera una URL concreta y devuelve un código de estado más un contenido decodificado y acotado.

Este es un paquete de **implementación**: registra un provider en `ctx.web`, no es dueño de la clave y no registra una herramienta orientada al modelo. Es un plugin de función/espacio de nombres (`inject: ['web']`).

## Reparto de responsabilidad

El provider es dueño de la **recuperación segura del recurso**: validación de URL, transporte HTTP, política de redirecciones, un timeout de respaldo del recurso, propagación del aborto, topes de bytes, decodificación de charset, clasificación del tipo de contenido y rechazo de binarios. `@deepseek-ai/dsh-tool-web` es dueño de la **presentación** (HTML→markdown, formato del truncamiento). Una respuesta HTTP no-2xx es un *resultado* (código de estado + cuerpo decodificado), no un error; `WebError` queda reservado para los fallos en recuperar o representar el recurso con seguridad.

El `timeoutMs` del provider es un respaldo de recurso para los llamadores directos de `ctx.web.fetch()` y los despliegues mal configurados, no el presupuesto de la llamada de herramienta orientada al modelo. [`dsh-tool-call-timeout-policy`](../../guard/timeout-policy/README.es.md) es dueño del presupuesto de la llamada de herramienta `web_fetch` armando `exec.signal`.

Un despliegue de herramientas web en producción fija el respaldo del provider por encima del presupuesto de la herramienta, así que las llamadas del modelo normalmente devuelven `TOOL_TIMEOUT`. Si el plazo externo alcanza al provider primero, el provider informa de `WEB_ABORTED` y la política externa lo sustituye por `TOOL_TIMEOUT`. `WEB_FETCH_TIMEOUT` identifica por tanto a un llamador directo del servicio cuyo presupuesto del provider se agotó.

## Higiene del transporte

- Acepta solo URLs `http:` y `https:`; rechaza credenciales en URLs (`WEB_BLOCKED_URL`) y URLs demasiado largas o malformadas (`WEB_INVALID_URL`).
- Aplica una longitud máxima de URL, un tope de bytes de respuesta (`WEB_FETCH_TOO_LARGE`), un tope de caracteres del cuerpo decodificado, un timeout (`WEB_FETCH_TIMEOUT`) y un tope de saltos de redirección.
- Propaga la señal de aborto del llamador (`WEB_ABORTED`) hacia la petición de red y la lectura en streaming.
- Sigue solo redirecciones del **mismo origen**; una redirección de origen cruzado falla con `WEB_REDIRECT_BLOCKED` y exige una llamada de herramienta nueva (el modelo del WebFetch de Claude Code).
- Envía un `User-Agent` de producto explícito, nunca un disfraz de navegador.
- Rechaza tipos de contenido no soportados (p. ej. binarios) con `WEB_UNSUPPORTED_CONTENT_TYPE`.

## Configuración

| Clave | Predeterminado | Significado |
|---|---|---|
| `maxUrlLength` | `2048` | Longitud máxima aceptada de la URL de petición. |
| `maxResponseBytes` | `5_000_000` | Tamaño máximo del cuerpo de respuesta en bytes. |
| `maxBodyChars` | `100_000` | Longitud máxima del cuerpo decodificado en caracteres. |
| `timeoutMs` | `30_000` | Timeout de fetch dentro del rango de temporizadores de Node — un respaldo de recurso para los llamadores directos de `ctx.web.fetch()`, no el presupuesto de la llamada de herramienta orientada al modelo (eso es `dsh-tool-call-timeout-policy`). |
| `maxRedirects` | `5` | Saltos máximos de redirección del mismo origen (`0` no sigue ninguna). |
| `userAgent` | `deepseek-harness/…` | Cabecera `User-Agent`. |

Los límites numéricos se validan en la construcción del plugin: todos los topes excepto `maxRedirects` deben ser un número finito positivo, y `maxRedirects` debe ser un entero no negativo. Un valor no válido lanza una excepción en lugar de construir silenciosamente un provider con límites sin sentido.

## Experiencia del modelo

Indirectamente, a través de [`dsh-tool-web`](../tool-web/README.es.md), que coloca el texto decodificado acotado por el `maxBodyChars` de este provider o el HTML con forma de markdown bajo su envoltorio de resultado de fetch y retiene los fallos del provider, mientras que las redirecciones, las cabeceras y la mecánica de transporte permanecen ocultas.

#### Efecto de KV Cache

Sin invalidación directa; el consumer nombrado es dueño de cualquier cambio en el prefijo de petición.

## Limitaciones conocidas y trabajo diferido

- **La protección SSRF / red privada está diferida** — no hay bloqueo de destinos privados, de loopback, link-local, multicast o de otro tipo no públicos; no hay resolver-DNS-para-luego-validar ni revalidación por salto (consulta la [Agent Note del seam de capacidad web](../../../.agents/notes/implemented/architecture/2026-06-24-web-capability-seam.es.md)). Hasta que aterrice, este provider es una primitiva SSRF y **no debe habilitarse** en un despliegue que pueda alcanzar destinos sensibles de la red interna.
- **Solo decodifica contenido textual** — las familias html/xhtml y `text/*`-más-JSON/XML; un `Content-Type` ausente o cualquier tipo binario lanza `WEB_UNSUPPORTED_CONTENT_TYPE`, y la decodificación de PDF con texto extraíble es trabajo diferido declarado.
- **El charset viene solo de la cabecera `Content-Type`** (UTF-8 por defecto) — una declaración `<meta charset>` de HTML se ignora, y una etiqueta de charset declarada pero no reconocida lanza una excepción en lugar de recurrir a un fallback.
