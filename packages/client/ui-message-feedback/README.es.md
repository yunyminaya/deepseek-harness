# @deepseek-ai/dsh-client-ui-message-feedback

[English](README.md) | [中文](README.zh.md) | Español

Plugin de feedback por mensaje, mitad navegador: un par Me gusta/No me gusta más una nota opcional, aportado como la entrada `feedback` (orden 10) de la tira `conversation.chat.assistant-actions`. La tira la declara `ui-conversation` y se renderiza dentro de la fila IconActions del mensaje de asistente finalizado, entre copiar y ramificar, así que los controles heredan el chrome y el comportamiento de hover de esa fila. El editor de notas en sí no está en esa fila: es un popover `role="dialog"` portado a `document.body` y anclado bajo su disparador, de modo que la fila mantiene su única línea tanto si el editor está abierto como cerrado y el panel no queda recortado por la columna de conversación. Un fallo de valoración o de carga de lista se muestra en línea en la fila; un fallo al guardar la nota se muestra dentro del popover, que permanece abierto para que el borrador pueda corregirse. Solo los mensajes finalizados llegan al slot: un parcial congelado por interrupción no lleva `messageId` y por tanto no tiene controles de feedback. La tira se renderiza una vez por turno, en el mensaje de asistente de cierre que es dueño de la fila IconActions del turno: los pasos anteriores de un turno de varios pasos producen filas de herramienta en lugar de un cuerpo valorable, así que no presentan controles aunque el Host los aceptaría como objetivos.

Un `MessageFeedbackController` por Session respalda cada control de mensaje de esa Session, así que una sola lectura de `messageFeedback.list` siembra todo el transcript. La lectura se difiere al primer hover o foco en lugar de dispararse al montar, porque los controles se montan una vez por mensaje asentado en el historial visible.

Las mutaciones pasan por `ctx.remote.messageFeedback`; el Host es dueño del compare-and-set por elemento. Cada `put` y `delete` lleva la `version` que este controlador observó por última vez, y una respuesta `version-conflict` trae el elemento autoritativo, así que una carrera perdida se reconcilia desde la propia respuesta en lugar de volver a obtener la Session. Las mutaciones se serializan por Session, de modo que una operación en cola siempre compara contra la versión confirmada. Volver a hacer clic en la valoración registrada retira el feedback; cambiar de lado arrastra la nota existente.

Los exports de `/client` son el cuerpo del plugin (`apply`/`inject`), el componente `MessageFeedbackActions`, la clase `MessageFeedbackController` y los tipos de cara inyectada.

## Experiencia de modelo

Ninguna, ya que el feedback es un sidecar que nunca entra en el registro de Session de solo añadido, en el contexto del modelo ni en la telemetría; ninguna valoración ni nota es jamás visible para el modelo.

#### Efecto en KV Cache

Ninguno; ninguna mutación de feedback toca la cola del historial.

## Limitaciones conocidas y trabajo diferido

- **El tamaño de la nota es una política del Host** — el despliegue configura `maxNoteBytes` (8192 en el bundle Web) y el Host rechaza una nota sobredimensionada con `note-too-large`. El editor no comprueba el límite de antemano, así que una nota sobredimensionada falla al guardar en lugar de mientras se escribe.
- **Sin push entre pestañas** — la valoración de una segunda pestaña se vuelve visible al reconectar o en la siguiente respuesta de conflicto, no inmediatamente; el sidecar no publica frames en vivo.
- **Solo vista de chat** — las vistas de trayectoria y waterfall no renderizan controles de feedback aunque sus nodos de asistente llevan ahora el mismo `messageId`.
