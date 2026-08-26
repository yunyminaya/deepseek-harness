# @deepseek-ai/dsh-session-title-first-prompt-llm

[English](README.md) | Español

Provider opcional de `ctx.sessionTitle` que resume el primer mensaje humano elegible a través de `ctx.llm`. Registra la cadencia `first-prompt`, se ejecuta automáticamente solo cuando una sesión nueva sin fork crea por primera vez su fallback, y atribuye el resultado al seq exacto de ese mensaje. Un fallo automático conserva el fallback y solo se reintenta mediante `ctx.sessionTitle.refresh()`.

El plugin usa la [configuración LLM compartida](../session-title-llm/README.es.md#configuration) completa que se requiere. Omite tanto `provider` como `model` para heredar la ruta exacta de la petición principal registrada en curso, o fija ambos para enrutar la generación de títulos de forma independiente.

## Model Experience

### Solicitud de título del primer mensaje

#### Lo que ve el modelo

El modelo de títulos recibe la instrucción de título compartida y un array JSON que contiene solo el primer mensaje humano elegible. Los prompts posteriores y el historial de fork heredado no disparan otra llamada automática.

#### Efecto en tokens

Como máximo se hace una petición auxiliar automática por sesión nueva, acotada por `maxInputBytes` y `maxOutputTokens`; los refrescos explícitos pueden hacer llamadas adicionales. La petición principal del agent no gana ningún token.

#### Efecto de KV Cache

Sin invalidación de la petición principal. La petición auxiliar usa la ruta configurada o registrada y tiene comportamiento de caché específico del provider.

## Limitaciones conocidas y trabajo diferido

- El primer mensaje por sí solo puede dejar de representar una sesión de larga duración; usa el provider de todos los mensajes cuando los prompts posteriores deban retitularla.
- Un fork conserva su título heredado y nunca ejecuta este provider automáticamente, ni siquiera cuando su primer mensaje sembrado provino del padre.
