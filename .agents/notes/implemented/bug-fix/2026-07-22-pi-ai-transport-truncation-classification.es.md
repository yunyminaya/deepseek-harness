# Agent Note: Clasificar los truncamientos de transporte de pi-ai a partir del texto del mensaje aplanado

Status: implemented

[English](2026-07-22-pi-ai-transport-truncation-classification.md) | Español

## Problema

Una ejecución de la TUI cuya conexión con el modelo se cortó a mitad de streaming mostraba el único aviso `terminated`, y una respuesta de Anthropic truncada mostraba `Anthropic stream ended before message_stop`. Ambos son truncamientos de transporte — la conexión murió antes del evento SSE terminal del provider — y sin embargo `classifyPiAiError` en `dsh-llm-pi-ai` no asignaba ninguno de los dos y caía en el comodín `PI_AI_ERROR`. Como `PI_AI_ERROR` no está en los `DEFAULT_RETRYABLE_CODES` de `llm-retry` (`RATE_LIMIT`, `SERVER`, `TIMEOUT`, `TRANSPORT`), un corte recuperable se trataba como fallo permanente y nunca se reintentaba.

La pérdida de detalle es aguas arriba e irrecuperable en el adaptador: pi-ai reduce un error capturado a `error.message` (`api/anthropic-messages.js`: `errorMessage = error instanceof Error ? error.message : JSON.stringify(error)`) antes de emitir el evento terminal `error`, descartando el `Error` original y su cadena de `cause`. undici transporta el `SocketError` accionable en `cause` pero entrega al envoltorio de fetch un simple `terminated`; pi-ai conserva solo esa palabra. Las `SimpleStreamOptions` de pi-ai no exponen ningún hook de fetch/dispatcher/client que permita capturar el `cause` antes de que se aplane.

## Decisión

- `classifyPiAiError` reconoce dos redacciones más de transporte y asigna ambas a `TRANSPORT`:
  - un corte de socket a mitad de streaming expresado como un `terminated` desnudo (undici) o `Premature close` (capa de streaming de Node);
  - un flujo truncado antes de su evento terminal, que cada provider de pi-ai lanza con su propia redacción (`Anthropic stream ended before message_stop`, `… before a terminal response event`, `… ended without a terminal event`, `Stream ended without finish_reason`), emparejada por `stream ended before/without`.
- El clasificador lleva una nota `XXX(pi-ai upstream)` que nombra el punto de aplanamiento y declara la corrección pretendida: clasificar por `code`/`cause` si pi-ai algún día reenvía el `Error` original o un hook que permita capturar el `cause`. Hasta entonces, la clasificación sigue siendo una equiparación de texto de mejor esfuerzo.
- `llm-pi-ai/README.md` gana una viñeta de limitaciones conocidas (Known-Limitations) que registra que pi-ai aplana la cadena de causa y que, por tanto, los códigos del harness se clasifican a partir del texto del mensaje.

La clasificación sigue basándose en el texto del mensaje porque esa es la única señal que pi-ai entrega; la marca `XXX` lo señala como una solución provisional, no como el estado final deseado.

## Alternativas consideradas

**Capturar el `cause` mediante un hook de fetch/dispatcher/client de pi-ai.** Rechazada: pi-ai 0.81.1 no expone ninguno. `StreamOptions` ofrece solo `onPayload`/`onResponse`; `onResponse` se dispara antes de que el cuerpo del flujo se consuma, así que no puede observar un corte a mitad de streaming. La ruta de Anthropic acepta un objeto `client`, pero construir e inyectar un cliente del SDK del provider por solicitud para interceptar errores de transporte rodea el límite del adaptador por una sola cadena de diagnóstico.

**Dejar ambos como `PI_AI_ERROR` y ampliar el conjunto reintentable de `llm-retry`.** Rechazada: `PI_AI_ERROR` es el comodín para fallos genuinamente sin clasificar, incluidos los no reintentables (una respuesta mal formada del provider, un fallo inesperado del SDK). Hacer reintentable el comodín reintentaría fallos que nunca van a triunfar; la corrección está en clasificar el caso recuperable, no en difuminar el cubo.

**Envolver el error aplanado en un `LlmError('TRANSPORT', { cause })` en el adaptador, a imagen del adaptador de DeepSeek.** Rechazada aquí: el adaptador de DeepSeek envuelve un rechazo de `fetch` *previo a la respuesta* cuyo `cause` sigue intacto, de modo que encadenar conserva el detalle real. En la ruta de pi-ai el `errorMessage` del evento terminal ya es una cadena aplanada sin `cause` que encadenar, así que envolver añadiría una capa sin recuperar nada; clasificar el código es el único valor que queda por añadir.

## Consecuencias

- Un corte de transporte a mitad de streaming y una truncación de flujo pre-terminal ahora llevan `TRANSPORT`, de modo que una política `llm-retry` compuesta los reintenta por defecto en lugar de fallar el turno.
- El texto del aviso no cambia (`terminated` / `Anthropic stream ended before message_stop`): el detalle de la causa se pierde antes de que el adaptador lo vea, así que `errorChain` no tiene nada más que renderizar. Solo mejoró el `code` enrutado.
- La clasificación sigue siendo equiparación de cadenas dependiente de la redacción del provider: una futura versión de pi-ai que reformule estos errores caería silenciosamente de vuelta a `PI_AI_ERROR` hasta que los patrones se actualicen. La nota `XXX` señala la corrección duradera (enrutar por un `code`/`cause` reenviado).
