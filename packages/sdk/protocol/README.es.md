# @deepseek-ai/dsh-sdk-protocol

[English](README.md) | Español

El wire protocol compartido para el runtime del SDK de DeepSeek Harness: una clase de transporte JSON-RPC 2.0 delimitado por nuevas líneas más los tipos con nombre de petición, resultado y notificación que hablan ambos extremos del wire. La raíz del paquete enumera la interfaz de consumidor del protocolo; los módulos de fuente no se exportan como deep imports. El lado servidor es el plugin [`dsh-sdk-jsonrpc-server`](../server/README.md); los clientes son [`dsh-sdk-client`](../client/README.md) (TypeScript) y el [SDK de Python](../../../python/README.md) (que refleja estas formas pero no las importa). Una librería pura — sin plugin, sin Config, sin registro.

## Transporte

`JsonRpcLineTransport` encuadra JSON-RPC 2.0 sobre flujos de bytes propiedad del caller, un frame JSON compacto por línea terminada en `\n`. Los frames con `id` y `method` son peticiones, `id` solo es una respuesta, `method` solo es una notificación; las líneas JSON malformadas se ignoran. `start()` adjunta listeners de flujo, `close()` los desadjunta y rechaza las peticiones pendientes sin destruir los flujos. Los handlers de petición ausentes responden `-32601`; los rechazos de handler responden `-32603` con el mensaje de error. Una respuesta de error rechaza el `request()` pendiente con `JsonRpcResponseError`, que conserva el `code` del wire y el `data` opcional. `JsonRpcTransportPeer` es la superficie de salida (request/notify) contra la que se tipa la clase del servidor.

## Tipos de wire

`types.ts` nombra cada payload del protocolo servido por `HarnessSdkJsonRpcServer`:

| Dirección | Método | Tipos |
|---|---|---|
| cliente→servidor | `initialize` | `InitializeParams` → `InitializeResult` |
| cliente→servidor | `session/prompt` | `SessionPromptParams` → `SessionPromptResult` (recibo de encolado durable) |
| cliente→servidor | `shutdown` | sin params → `{}` |
| servidor→cliente | `session.event` | `SessionEventNotification` (cada sesión del runtime, sin filtrar) |
| servidor→cliente | `session.status` | `SessionStatusNotification` (transición `running`/`idle` de todo el agent) |
| servidor→cliente | `subagent.started` | `SubagentStartedNotification` |
| servidor→cliente | `subagent.finished` | `SubagentFinishedNotification` (solo runs en proceso) |

`HarnessSdkRequestMap` y `HarnessSdkNotificationMap` los indexan por nombre de método. `SessionPromptResult.messageId` identifica el `UserMessage` encolado; no identifica un mensaje de assistant posterior, el fin de turno ni un resultado de prompt. Los clientes combinan el flujo abierto `session.event` con el `session.status` de todo el agent según su propia propiedad de actividad. `SubagentFinishedNotification.lastAssistantMessage` contiene el último mensaje de assistant no vacío del hijo o, cuando no existe tal mensaje, su texto de assistant acumulado; el campo está ausente cuando el hijo no produjo ninguno de los dos. `InitializeParams.maxTokens` es un entero seguro positivo opcional que limita la salida de cada modelo de conversación para los agents creados por SDK y sus descendientes en proceso; omitirlo permite que se aplique el default del modelo exacto del adaptador seleccionado o, si no, conserva el comportamiento del provider. Los tipos de payload de notificación dependen de `SessionEvent` (`dsh-session`), `ContentBlock` (`dsh-llm`) y `SubagentStopReason` (`dsh-subagent`) — el protocolo transmite sobres completos de session log, así que el vocabulario de sesión forma parte del contrato de wire. `serverInfo.name` sigue siendo el estable de wire `deepseek-harness-sdk-runtime`.

## Experiencia del modelo

Ninguna, ya que este paquete define el wire protocol orientado al cliente; las superficies visibles para el modelo pertenecen a los plugins de runtime compuestos detrás de la entrada [`dsh-sdk-jsonrpc-server`](../server/README.md) que sirve.

#### Efecto en la caché KV

Ninguno; este paquete ni ensambla ni envía una petición de provider.

## Limitaciones conocidas y trabajo diferido

- **Sin negociación de versión de protocolo** — el handshake lleva solo `serverInfo.version` (`0.0.1`, sin validar por los clientes); postura de pre-release, sin promesa de compatibilidad.
- **Sin métodos de cancelación ni de cierre de sesión** — un cliente abandona un turno cerrando el proceso del runtime; consulta el [README de `dsh-sdk-jsonrpc-server`](../server/README.md).
- **Las peticiones servidor→cliente son capacidad muerta** — el transporte las soporta, pero el servidor nunca envía una; la superficie de responder del SDK de Python existe para futuros flujos de aprobación.
