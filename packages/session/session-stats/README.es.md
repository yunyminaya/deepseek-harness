# @deepseek-ai/dsh-session-stats

[English](README.md) | [中文](README.zh.md) | Español

Plugin de función que registra la unidad de proyección `sessionStats`: cifras de conversación de todo el log — recuentos de turnos/pasos y los tiempos de pared de LLM (modelo de lenguaje de gran tamaño), herramienta, primer token y decode — plegado desde los límites de paso, los chunks de stream, los pares de herramientas y los mensajes de assistant ensamblados, y servido a través del seam de proyección de sesión (instantánea del registro, feed de cambios y cada carrier de proyección: página de cola del historial, tramas push `session/projection`, filas de la lista de sesiones). Los clientes renderizan cifras de sesión completa que la paginación y la compactación no pueden cambiar; el consumidor de referencia es la franja de estadísticas del chat web, cuyo pliegue de ventana refleja estos nombres de campo como su respaldo sin unidad.

## Semántica de pliegue

- `steps` cuenta los eventos `step/end`. El agent loop (bucle del agente) añade exactamente uno por cada paso entrado, en un `finally`, de modo que los pasos completados, fallidos, cancelados y por max-tokens cuentan todos. Contar los mensajes de assistant ensamblados en su lugar sobrecontaría los mensajes de usage-host por max-tokens (contenido vacío, excluidos de la superficie) e infracontaría los pasos cancelados (abortados antes de que el mensaje se ensamble).
- `turns` cuenta turnos distintos con al menos un paso cerrado; los turnos rechazados o vacíos (cerrados sin paso) no se cuentan. Los números de turno los asigna el host y son monótonos por sesión, así que el pliegue conserva solo el último turno contado.
- `llmMs` suma `step/start` → `assistant/message` por cada paso que ensambló un mensaje (las esperas de reintento dentro del paso son tiempo de modelo, como en el pliegue de ventana).
- `ttftMs`/`ttftSteps` suman y cuentan `step/start` → primer chunk delta no vacío; el límite del primer intento sobrevive a un `llm/retry` dentro del paso (paridad con `resetForRetry` de la ventana).
- `decodeMs`/`decodeTokens` suman el primer token → mensaje ensamblado y los tokens de salida informados por el provider, solo sobre pasos que tengan ambos.
- `toolMs` suma los pares `tool/call` → `tool/result` emparejados por callId; las llamadas sin resolver se descartan en `turn/end` (los resultados aterrizan dentro de su turno).
- Cada campo es 0 hasta su primer evento contribuyente. Un registro compuesto siempre sirve la clave, así que los clientes leen el valor, nunca la presencia de la clave.

## Composición

```yaml
- id: session-stats
  name: '@deepseek-ai/dsh-session-stats'
```

Inyecta `sessionProjections` — el propósito completo del plugin; en ensamblajes sin el registro la fibra queda pendiente y no se registra nada.

## Experiencia del modelo

Ninguna, ya que el plugin solo computa un modelo de lectura orientado al cliente de los eventos de sesión ya registrados y no toca ningún prompt, mensaje, schema, stream ni resultado de herramienta.

#### Efecto de KV Cache

Ninguno; el plugin nunca ensambla ni envía solicitudes de provider.

## Limitaciones conocidas y trabajo diferido

- **Los pasos cuentan trabajo intentado, no salida visible** — un paso que falló antes de producir contenido visible igualmente se cerró con `step/end` y cuenta; un paso interrumpido por un crash cuenta después de que la sesión se recargue, cuando la recuperación por crash añade su `step/end` sintético (`interruptedTurnClosers` en dsh-session).
- **Un paso cancelado se cuenta pero no se cronometra** — no se ensambla ningún mensaje de assistant, así que su tiempo parcial de stream no entra en ninguna cifra de tiempo de pared, en consonancia con el nodo interrumpido sin cronometrar del pliegue de ventana; un mensaje de usage-host por max-tokens contribuye a la inversa tiempo de modelo que la superficie no muestra.
- **Los recuentos están acotados al log, no a la superficie** — los pasos cuyos mensajes se compactaron después siguen contados; las cifras describen la sesión completa, no la superficie actual visible al modelo.
- **Montado solo en el bundle de la web-app** — otros ensamblajes no sirven ninguna clave `sessionStats`, y sus consumidores recurren al conteo acotado a la ventana (la ruta de respaldo de la franja de estadísticas web).
