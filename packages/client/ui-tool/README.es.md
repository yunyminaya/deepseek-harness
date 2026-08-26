# @deepseek-ai/dsh-client-ui-tool

[English](README.md) | Español

Plugin de presentación de Tool del cliente. `ui-conversation` despacha cada Nodo de Conversación `tool-call` ordenado a través de la clave correspondiente de `conversation.chat.node`; este paquete renderiza su raíz y los hijos de Code Dispatch, y luego despacha cada llamada atómica a través del slot con clave `tool.call.toolview`. Los nombres de Tool no registrados usan la tarjeta genérica.

Los paquetes de UI de negocio registran solo sus nombres de Tool de cable y sus vistas atómicas. No emparejan eventos de Session, no reconstruyen el transcript ni son dueños de la topología raíz/subllamada. El Runtime sigue siendo la autoridad para el emparejamiento llamada/resultado, el ciclo de vida y la proyección recursiva de `subCalls`; la vista de conversación sigue siendo la autoridad para la colocación en ChatFlow.

## Contrato de renderizado

`ToolCallTree` recibe una única `ToolCallBlock` raíz que ya contiene los `subCalls` recursivos, el estado de selección, el `cwd` de la sesión y callbacks del Host para abrir archivos e inspeccionar llamadas. Recorre recursivamente los bloques de llamada estándar y envía la raíz y los hijos a cada profundidad por la misma ruta de despacho atómico, sin suscribirse a un mapa separado de padre a hijos.

Cada envoltorio de raíz e hijo conserva el contrato DOM `data-chat-anchor-key="call:<id>"` y `data-chat-call-id` usado para paginar y seleccionar.

El paquete también llena `conversation.details.tool` con `ToolDetails`. Los renderizadores de fila y de detalle comparten los mismos modelos de tarjeta puros para las intenciones de renderizado `terminal`, `read`, `diff`, `search` y `web`. Las etiquetas de intención desconocidas y los datos de tarjeta de cable malformados caen al texto aplanado del resultado de la Tool.

Las filas genéricas clasifican los nombres de Tool conocidos en variantes de search, read, shell, write, edit, code o genérica. Los estados de ciclo de vida en ejecución, con éxito, fallido e interrumpido provienen solo de la porción congelada llamada/resultado. Las rutas de archivo se resuelven contra el `cwd` de la sesión solo cuando el usuario invoca el callback del Host de abrir archivo; el código de presentación no lee servicios de Session.

## Vistas atómicas de Tool

Un paquete de negocio propietario registra su nombre de Tool de cable en `tool.call.toolview`:

```ts ignore-check
ctx.slots.inject('tool.call.toolview', () =>
  ctx.slots.register({
    name: 'tool.call.toolview',
    key: '<wire tool name>',
  }, BusinessToolRow))
```

El payload del propietario es `ToolCallOwnerProps`: `callId`, `toolName`, el `block` congelado, `cwd` y `home` opcionales, y callbacks simples `openFile`/`inspect`. Los resúmenes de ruta se relativizan primero contra el cwd de la sesión y después reemplazan un home POSIX sobrante con `~`; `filePath` y el open del Host conservan la ruta de archivo original del sistema de archivos. El registro recibe la porción normal de slot runtime de sesión. No recibe nodos React, servicios del Runtime ni conocimiento de raíz/subllamadas.

Este paquete es dueño hoy de la alternativa genérica y de las presentaciones integradas de shell/pwsh, read, write/edit, grep/glob, web, todo, question y Code Dispatch. `ui-skill` demuestra un registro de negocio para `skill`.

Los límites y las reglas de alternativa específicos de cada tarjeta siguen en las notas del [terminal](../../../.agents/notes/implemented/feature/2026-07-28-web-terminal-card.es.md), [diff](../../../.agents/notes/implemented/feature/2026-07-30-web-diff-card.es.md), [read](../../../.agents/notes/implemented/feature/2026-07-30-web-read-card-frontend.es.md), [search](../../../.agents/notes/implemented/feature/2026-07-30-web-search-card.es.md) y [web](../../../.agents/notes/implemented/feature/2026-07-30-web-result-card-frontend.es.md).

## Experiencia del modelo

Ninguna, ya que este paquete renderiza llamadas y resultados de Tool ya registrados sin alterar solicitudes de modelo, ejecución de Tool ni eventos de Session.

#### Efecto en la caché KV

Ninguno. El paquete es solo presentación de cliente.

## Limitaciones conocidas y trabajo diferido

- El Host excluye `run_code` de los enlaces de programa de Code Mode, así que los eventos de producción producen un nivel de despacho; el contrato recursivo Runtime/UI soporta el anidamiento.
- Las vistas de Tool de primera parte están colocadas aquí y pueden moverse a sus paquetes de negocio propietarios de forma independiente a través del slot con clave.
- Las copias de Tool reutilizan el namespace de locale de `ui-conversation`.
