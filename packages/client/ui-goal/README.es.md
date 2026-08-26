# @deepseek-ai/dsh-client-ui-goal

[English](README.md) | Español

Plugin de superficie de objetivo, mitad de navegador: la franja `GoalBar` es la segunda tarjeta independiente de la pila de contexto del composer `conversation.input.dock` (orden 10, después de Todo y antes de Queue). El objetivo en vivo llega a través de `useProjection('goal')` —el valor completo calculado por el Host, sembrado por la página de cola de historial y actualizado por los frames de `session/projection`—, así que el plugin no posee ningún store de dominio, cadena de refresco ni listener de eventos. La cara de inyección del slot solo lleva los cuatro verbos de mutación (edit / pause / resume / clear a través de `ctx.remote.goals` —un objetivo activo ofrece la acción de pausa y uno en pausa la de reanudar—); cada uno lee la ref CAS del valor proyectado actual de la sesión en el momento de la llamada y muestra en línea el error de Remote rechazado. La franja ejecuta las mutaciones una a una y de forma síncrona porque el render pendiente de React no puede impedir los clics del mismo frame; tras un clear correcto, suprime de inmediato ese id de objetivo exacto mientras la proyección nula autoritativa se pone al día. La creación de objetivos sigue en el comando `/goal` del Host; los objetivos en carga, ausentes, completados o borrados con éxito no renderizan nada.

El plugin proyecta por separado cada `command/run` de `/goal` duradero a través de su propia Conversation Definition. Construye un Chat Node `command-input` antes del Node de resultado de comando genérico y registra el renderizador con clave de ese Node como una burbuja de estilo usuario monoespaciada, alineada a la derecha, de 14px/22px, con el nombre de grupo localizado `Command input` / `命令输入` y sin marca de tiempo, copia ni acciones de bifurcación. El Node visible que no es de comando activa Chat en fresco; una recarga lo reconstruye a partir de la ejecución, mientras que una ventana de historial que solo contiene `command/done` conserva únicamente la fila de resultado genérica. Esta proyección nunca crea `user/message` ni un turno de modelo.

Los exports de `/client` son el cuerpo del plugin (`apply`/`inject`), los componentes `GoalBar`/`GoalDock` y los tipos de cara de verbos inyectados.

## Experiencia del modelo

De forma indirecta, a través de los métodos Remote `goals/edit`, `goals/pause`, `goals/resume` y `goals/clear` que invoca la franja: cada mutación aceptada se confirma en una inserción duradera `agent/inbox/spliced`, que la proyección de objetivo pliega de inmediato, y pone en cola un mensaje de contexto `goal/change`. El modelo solo ve ese contexto si un pre-paso posterior lo admite; descartar el mensaje encolado no revierte el estado proyectado. La franja en sí no añade ningún contenido de prompt.

#### Efecto en la KV Cache

Ninguno salvo que se admita el contexto de objetivo encolado. Un contexto admitido extiende la cola del historial como cualquier otro mensaje; una inserción descartada antes de la admisión no afecta a la caché.

## Limitaciones conocidas y trabajo pendiente

- **Solo fase duradera** — la proyección omite la activación local del proceso, así que la franja no puede distinguir un objetivo activo pero desarmado de uno armado; resume vuelve a armar por el lado RPC. No existe ningún canal de activación en vivo del Host.
