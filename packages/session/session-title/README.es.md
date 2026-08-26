# @deepseek-ai/dsh-session-title

[English](README.md) | Español

Títulos de sesión respaldados por el log con un fallback determinista inmediato y un provider asíncrono opcional. Cada revisión aceptada es un evento `session/title` solo de registro; `foldSessionTitle()` y `ctx.sessionTitle.get()` seleccionan el evento más reciente y devuelven su seq de evento y su marca de tiempo.

Solo los bloques de texto de los eventos `user/message` humanos son elegibles. El primer prompt elegible programa un fallback a partir de sus primeras palabras dentro del límite de bytes UTF-8 configurado. El espacio en blanco se normaliza, las secuencias de control de terminal se eliminan y el truncamiento nunca divide un code point. Los prompts vacíos y no textuales esperan a una entrada elegible posterior.

## Service: `SessionTitleService` (ctx key: `sessionTitle`)

- `get(session)` reduce el último título aceptado desde un log en vivo o reproducido.
- `refresh(session, signal?)` materializa el fallback cuando es necesario y luego ejecuta explícitamente el provider registrado sobre los mensajes elegibles en curso. Los errores del provider y la cancelación del llamador rechazan; la cancelación no revierte un evento de fallback ya aceptado.
- `rename(session, title)` acepta un título de usuario explícito de forma síncrona: normaliza el texto, sustituye el trabajo automático en curso y anexa un evento `session/title` con la fuente `user`. Un último título de fuente de usuario fija la sesión: los mensajes de usuario posteriores no programan ninguna revisión automática; un `refresh` explícito sigue siendo el modo deliberado de desfijarla.
- `register(provider)` instala el único provider opcional y devuelve su disposer de efecto Cordis awaitable. Un segundo registro lanza una excepción de inmediato; la disposición aborta las llamadas pendientes y activas, espera a que se resuelvan y solo entonces permite que se registre otro provider.

El trabajo automático nunca retrasa la respuesta principal del agent. Un provider solo arranca después de que la ruta exacta de una petición marcada construida por el loop coincida con el `request/header` registrado en curso, incluido el caso en que la cabecera sin cambios no necesita una instantánea nueva. Su finalización tardía anexa un evento independiente solo de registro directamente a través de `Session` sin abrir un turno. La persistencia observa ese evento de forma eager y drena en los checkpoints ordinarios del ciclo de vida; la publicación del título en sí no fuerza un flush. Los fallos automáticos avisan y conservan el último título. Las revisiones nuevas de todos los mensajes, la disposición del provider, la disposición de la sesión y el refresh explícito abortan el trabajo anterior, y una finalización obsoleta no puede anexar. Los refrescos explícitos concurrentes reservan su revisión antes del trabajo del provider, mientras que las peticiones de fallback automáticas y explícitas solapadas comparten una anexión en curso local a la sesión. El servicio y el provider de modelo incluido anexan cada uno su propio tipo de evento literal, así que no se necesita marcador genérico de escritura de título, cast ni cola de resolución. El teardown del servicio cancela el trabajo en cola y drena las llamadas que ignoran la cancelación antes de que la descarga (unload) se complete.

Los forks heredan los eventos de título en su seed sin cambios. La cadencia first-prompt no retitula automáticamente a un hijo; la cadencia all-messages puede anexar una revisión nueva después de que el hijo reciba un prompt humano posterior.

## Configuración

Todos los límites son obligatorios; la librería no suministra valores predeterminados.

| Clave | Contrato |
|---|---|
| `fallbackMaxWords` | Máximo positivo de palabras delimitadas por espacio en blanco en el fallback determinista. |
| `fallbackMaxBytes` | Máximo positivo de bytes UTF-8 en el fallback; no debe superar `maxTitleBytes`. |
| `maxTitleBytes` | Máximo positivo de bytes UTF-8 aceptados de cualquier fuente. |

## Contrato del provider

Un provider aporta un id estable con brand, un modo automático (`first-prompt` o `all-prompts`) y `generate(request)`. La petición transporta la sesión en vivo, todos los mensajes elegibles a lo largo de una revisión fija, la ruta de petición principal registrada en curso cuando está disponible y la cancelación. El resultado identifica un título no vacío, los seq únicos y ordenados de los mensajes de origen de esa petición, y la ruta provider/model opcional usada para generarlo. El servicio normaliza y valida el resultado antes de que se vuelva duradero.

Consulta las [estructuras de datos de títulos de sesión](../../../docs/subsystems/session-title.es.md) y la [decisión implementada](../../../.agents/notes/implemented/feature/2026-07-21-log-backed-session-titles.es.md).

## Model Experience

### Estado del título de sesión

#### Lo que ve el modelo

Nada. `session/title` es solo de registro y nunca entra en la surface de la sesión, en `deriveMessages()`, el prompt de sistema, los schemas de herramientas ni el prefijo de petición.

#### Efecto en tokens

El fallback y las revisiones de provider aceptadas añaden cero tokens a la petición principal del agent. La petición auxiliar independiente de un provider opcional la documenta el propio paquete del provider.

#### Efecto de KV Cache

Ninguno para la petición principal; los eventos de título no cambian su contenido reconstruido ni su clave de caché.

## Limitaciones conocidas y trabajo diferido

- La eliminación de títulos (volver a desfijar hacia títulos automáticos sin un `refresh` explícito), la búsqueda y la indexación de listados quedan fuera de este servicio.
- El registro de providers acepta deliberadamente como máximo una implementación, de modo que un despliegue no puede componer estrategias de título en competencia sin escribir un provider que sea dueño de su precedencia.
