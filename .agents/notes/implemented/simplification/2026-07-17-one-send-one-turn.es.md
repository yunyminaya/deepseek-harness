# Agent Note: Eliminar el batching implícito de los envíos ordinarios

Status: implemented

[English](2026-07-17-one-send-one-turn.md) | Español

## Problema

Supongamos que un llamador envía el mensaje A y luego el mensaje B con dos llamadas `Agent.send()`. El batching implícito puede poner A y B en un solo turno simplemente porque ambos están esperando cuando el driver lee su cola. El llamador hizo dos llamadas, pero el loop las convierte silenciosamente en una sola unidad de trabajo.

Esa agrupación depende de la temporización, no de la intención del llamador. Las llamadas de una misma pila síncrona, microtasks vecinas, listeners de eventos y callbacks de modelo podrían agruparse de forma distinta aunque todos los llamadores usaran la misma API.

Esta agrupación cambia el comportamiento, no solo el número de llamadas al modelo. Un turno ordinario es propietario de un follow-up reclamado, `turn/start`, `turn/end` y un checkpoint de durabilidad. Si el mensaje B comparte el turno de A, B puede entrar en el request de modelo de A en lugar de ver primero el resultado cerrado de A en el log de sesión. Entrar en un follow-up mientras se rechaza otro también exige un estado mixto que ningún llamador pidió.

## Decisión

Cada `send()` exitoso crea un elemento de cola FIFO independiente. Si ese elemento se ejecuta, es el único mensaje ordinario en su turno. Un elemento puede descartarse antes de empezar, así que la garantía precisa es como máximo un turno en lugar de exactamente uno; dos envíos nunca se combinan silenciosamente.

Antes de insertar un mensaje, `send()` comprueba el estado del agente y acepta un valor ya identificado y congelado en profundidad. El splice durable y `agent/inbox/inserted { message }` conservan su `MessageId`; el mensaje pendiente sigue siendo direccionable mediante `Inbox.replace()` e `Inbox.remove()` hasta que el driver lo reclame o lo descarte. El [decidido lifecycle de inbox pre-step reclamado](../architecture/2026-07-31-claimed-pre-step-inbox-lifecycle.es.md) es propietario del ciclo de vida actual.

Si los mensajes A y B se procesan ambos, el turno de B solo empieza después de que A registre `turn/end` y el checkpoint de durabilidad de A se asiente. El request de B ve por tanto cualquier resultado cerrado que A haya dejado en el mismo log de sesión. Un error de checkpoint se informa, pero el settlement solo libera esta barrera de orden; no hace durable una escritura fallida. Un `cancel()` amplio, la destrucción o un fallo antes de `turn/start` pueden en cambio descartar un elemento no iniciado sin abrir un turno vacío.

En un límite de turno, el loop abre el turno y reclama un follow-up después de la entrada de siguiente-paso pendiente. `agent/pre-step` o rechaza la propuesta o devuelve el lote de entrada completo. Un follow-up rechazado permanece eliminado y cierra un turno sin-paso bloqueado sin escribir historial visible para el modelo. Las ramas mixtas de follow-up ordinario no existen.

La regla de no-batching se aplica solo a la entrada ordinaria de follow-up. `steer()` pone la entrada en el inbox de siguiente-paso y despierta al driver. Durante un turno, el loop puede reclamarla en un límite de paso posterior; mientras está inactivo, el lote de siguiente-paso que despierta empieza un turno nuevo. La entrada que llega después de que un lote fuera reclamado espera un límite posterior, mientras que la cancelación o la destrucción pueden descartarla.

`inject()` sigue añadiendo contexto orientado al modelo sin enviar entrada ordinaria ni despertar al driver. Siempre espera en el inbox de siguiente-paso un pre-step posterior, incluso estando inactivo; AgentLoop la registra como `user/message` solo cuando una decisión de entrada la devuelve dentro de un turno. `cancel()` sigue siendo una operación de agente completo que puede limpiar toda la entrada ordinaria no iniciada, el steering y la inyección y abortar el paso actual. `status` y `whenIdle()` también describen al agente completo, no a un mensaje.

## Alternativas consideradas

**Conservar el batching automático de envíos ordinarios para reducir las llamadas al modelo.** Esto puede mejorar el rendimiento cuando los productores superan al driver, pero hace que los límites de turno dependan de la planificación y permite que un mensaje posterior se ejecute antes de que el turno precedente se cierre y alcance su checkpoint. La decisión conserva el límite predecible y acepta las llamadas extra. Cualquier funcionalidad futura de batching necesita un contrato explícito visible para el llamador respaldado por mediciones.

## Verificación

- Las pruebas unitarias y de propiedades envían desde la misma pila, microtasks vecinas, productores distintos y callbacks reentrantes; cada mensaje recibe su propio turno ordenado FIFO.
- Un test de stdio compilado envía dos líneas y observa dos requests al modelo y dos límites de turno.
- Los checkpoints de primer turno retrasados y rechazados mantienen esperando al siguiente turno y prueban que su request ve el resultado de asistente precedente.
- Las pruebas de la vía de fallo cubren el rechazo de pre-step, el fallo de listener, la cancelación amplia, la destrucción y el fallo antes de `turn/start`; las salidas de pre-step iniciales cierran turnos sin-paso balanceados, los mensajes no se fusionan y el trabajo posterior superviviente sigue drenándose.
- Pruebas separadas cubren `steer()` de turno abierto, turno fallido e inactivo, `inject()` pendiente, el status de agente completo y `whenIdle()`.

## Consecuencias

Los límites de turno ordinarios son predecibles: los mensajes A y B permanecen separados, y B solo se ejecuta después de que A se haya cerrado y alcanzado su checkpoint. Los llamadores siguen sin recibir un handle de completado por envío; un mensaje pendiente puede eliminarse mediante su `MessageId`, la cancelación amplia puede descartar toda la cola no iniciada, y el status y la quietud siguen siendo observaciones de agente completo.

El intercambio son más requests al modelo y más checkpoints. Una cola ocupada puede tardar más en drenarse y puede crecer bajo productores sostenidos. El batching de envíos ordinarios solo vuelve mediante un contrato explícito y medido.
