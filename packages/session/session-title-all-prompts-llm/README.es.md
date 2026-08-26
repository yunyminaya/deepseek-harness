# @deepseek-ai/dsh-session-title-all-prompts-llm

[English](README.md) | Español

Provider opcional de `ctx.sessionTitle` que resume cada mensaje humano elegible a través de `ctx.llm`. Registra la cadencia `all-prompts` e inicia una revisión nueva después de cada prompt humano nuevo, usando el historial sembrado además de los prompts de sesiones hijas. Una revisión más reciente aborta y sustituye el trabajo anterior; incluso un provider que ignora la cancelación no puede confirmar salida obsoleta.

El plugin usa la [configuración LLM compartida](../session-title-llm/README.es.md#configuration) completa que se requiere. Omite tanto `provider` como `model` para heredar la ruta exacta de cada petición principal registrada en curso, o fija ambos para enrutar la generación de títulos de forma independiente. Si el prompt agregado enmarcado final supera `maxInputBytes`, la petición falla en lugar de truncar el historial; el uso automático avisa y conserva el título anterior.

## Model Experience

### Solicitud de título de todos los mensajes

#### Lo que ve el modelo

El modelo de títulos recibe la instrucción de título compartida y un array JSON de todos los mensajes humanos elegibles hasta la revisión en curso, en orden de log con los seq exactos. El historial sembrado se incluye.

#### Efecto en tokens

Una petición auxiliar puede seguir a cada prompt elegible nuevo, acotada por petición mediante `maxInputBytes` y `maxOutputTokens`; los refrescos explícitos pueden añadir llamadas. La petición principal del agent no gana ningún token.

#### Efecto de KV Cache

Sin invalidación de la petición principal. La entrada auxiliar crece o cambia después de cada prompt, así que la reutilización de caché específica del provider termina en el primer token JSON cambiado.

## Limitaciones conocidas y trabajo diferido

- El desbordamiento de entrada conserva el título anterior; este provider no tiene política de resumen-de-resúmenes ni de retención para sesiones muy largas.
- Trata todos los mensajes humanos elegibles por igual y no ofrece ponderación, filtrado ni precedencia de títulos manuales.
