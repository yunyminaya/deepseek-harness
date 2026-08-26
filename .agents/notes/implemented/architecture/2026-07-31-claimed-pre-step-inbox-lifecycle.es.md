# Agent Note: Reclamar la entrada del inbox antes de una decisión de pre-step

Status: implemented

[English](2026-07-31-claimed-pre-step-inbox-lifecycle.md) | Español

## Problema

El loop dividía antes un solo límite de paso entre la preparación del prompt, la admisión del prompt y un hook de paso en serie. Un resultado de admisión podía retener o descartar la entrada reclamada, y los eventos en vivo de la cola llevaban formas que duplicaban el estado durable del inbox. Los plugins tenían que elegir entre mutar el inbox, reescribir un lote enviado o anexar directamente al historial de la sesión, mientras que los observadores no podían depender de un orden exacto.

Los wrappers de inbox locales a la ocurrencia también duplicaban la identidad que ya llevaba cada `UserMessage`. Convertían la inserción, la edición, el reclamo, la cancelación, la proyección de reconexión y la entrada al paso en un único protocolo combinado, aunque la sesión de solo añadido ya era dueña de la proyección durable de la cola.

## Decisión

Antes de cada paso propuesto, `Inbox.claim(target)` elimina atómicamente el lote completo: todos los mensajes `next-step` y, en un límite de turno, un mensaje `next-turn`. En el límite inicial, el loop confirma primero `turn/start`, de modo que el reclamo y su única decisión `agent/pre-step` tienen propiedad durable del turno. El reclamo registra eliminaciones puras normalizadas `agent/inbox/spliced` sin resultado. El loop emite entonces `agent/inbox/claimed { message, turn }` una vez por mensaje reclamado y espera el waterfall con ese lote exclusivo y `{ turn, step, signal }`.

`PreStepDecision` es `{ kind: 'reject' } | { kind: 'enter'; messages: UserMessage[] }`. Reject no abre ningún paso, deja el lote reclamado eliminado y cierra el turno como bloqueado sin eventos de paso. La entrada vacía, la cancelación y el fallo antes de `step/start` cierran igualmente un turno equilibrado sin pasos. Enter entrega el lote completo anexado como eventos `user/message` después de `step/start`. Un listener que envuelve `next()` conserva los cambios posteriores salvo que los reemplace intencionalmente, de modo que todas las reescrituras de mensajes se asientan una sola vez en el valor de retorno final. No existe ningún punto de extensión `agent/prompt-prepare`, `agent/prompt-submit` ni `agent/step`.

El inbox durable sigue siendo dos listas `UserMessage[]` direccionadas por `MessageId`. `append`, `prepend` y `splice` toman un destino, mientras que `replace(messageId, newMessage)` y `remove(messageId)` localizan el mensaje pendiente en ambas listas antes de confirmar un splice normalizado. El reemplazo puede cambiar la identidad y emite el mensaje antiguo como descartado seguido del mensaje nuevo como insertado. Cada inserción emite `agent/inbox/inserted { message }`; una eliminación ordinaria registra `outcome: 'canceled'` y emite `agent/inbox/discarded { message }`. El reclamo es la operación interna del loop sobre el inbox en el límite de paso y registra eliminaciones puras sin notificaciones ni resultado, de modo que el propio loop puede publicar los eventos de reclamo. Estos eventos en vivo no añaden campos de colocación, resultado ni lote.

Las dos superficies de eventos tienen consumidores separados. Los observadores que siguen un solo mensaje usan `agent/inbox/inserted`, `claimed` y `discarded`. Los consumidores de toda la cola, incluida la proyección Web de la cola y la línea base de reconexión, usan el stream durable `agent/inbox/spliced`; las ediciones y eliminaciones de la UI se enrutan por `Inbox.splice()` u otro método de mutación de Inbox, de modo que la misma proyección registra cada cambio.

Los plugins que necesitan una reescritura atómica del paso actual devuelven mensajes desde `agent/pre-step`. Los plugins que solo necesitan contexto posterior pueden mutar `agent.inbox` directamente. El contexto de workspace usa ambas vías: las proyecciones asíncronas del sistema de archivos preparan un único elemento `next-step` reemplazable, mientras que el siguiente pre-step entrante pliega ese elemento —o una línea base recién compuesta— en su lote final y elimina la copia pendiente. El rechazo mantiene el elemento en cola.

La [decisión archivada sobre la cola direccionable por ocurrencia](../../archived/feature/2026-07-29-addressable-queue-operations.md) describe el diseño de wrapper de ocurrencia superado. `MessageId` es ahora dueño del direccionamiento, mientras que el mirror conservado de la cola del Host deriva sus instantáneas de la proyección durable de splices.

## Alternativas consideradas

**Conservar hooks de preparación y admisión separados.** Esto permite que la preparación mute el inbox antes del reclamo y que la admisión reescriba después, pero crea dos superficies de orden para un solo límite y hace ambigua la propiedad de la cancelación.

**Dejar que el rechazo vuelva a poner en cola el lote reclamado.** Esto conserva un comportamiento similar al reintento, pero convierte un veto en una mutación oculta de la cola, duplica el trabajo posterior salvo que cada carrera quede cercada, e impide que el reclamo sea una transferencia atómica de propiedad.

**Poner colocación y resultado en cada evento en vivo.** Los splices durables ya son dueños de esos hechos. Repetirlos en las notificaciones en vivo crea un segundo contrato que puede desviarse y es innecesario para los consumidores que ya tienen la identidad exacta del mensaje.

## Verificación

La cobertura del agent loop fija el orden turn-start-antes-de-reclamo-antes-de-pre-step, las cargas útiles exactas de los eventos en vivo, el rechazo equilibrado sin pasos, la reescritura del lote final, la entrada insertada tras un reclamo, el fallo del listener y la cancelación. Los tests de Inbox y de los consumidores fijan las eliminaciones puras de reclamo, las eliminaciones ordinarias canceladas, la preparación de agent-instructions, el reemplazo y la entrada en el mismo paso, el comportamiento de plan/goal/hook, la limpieza de la UI, la compactación, el checkpointing y la proyección durable reanudada. Los catálogos generados de eventos y tipos exponen solo el nuevo waterfall y las cargas útiles.

## Consecuencias

El loop tiene una única decisión esperada antes de cada paso y una sola transferencia de propiedad para su entrada. Los mensajes reclamados nunca vuelven al inbox implícitamente; las inserciones posteriores siguen siendo independientes. Los eventos en vivo son simétricos a las demás notificaciones del inbox sin reflejar los metadatos durables, y los plugins pueden elegir explícitamente entre la reescritura exacta del paso actual o la entrega ordinaria posterior al inbox.
