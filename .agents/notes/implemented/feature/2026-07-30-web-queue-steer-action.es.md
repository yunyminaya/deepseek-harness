# Agent Note: Dirigir un mensaje Web en cola hacia el turno activo

Status: implemented

[English](2026-07-30-web-queue-steer-action.md) | Español

## Problema

El composer Web encolaba originalmente cada envío con Enter mientras corría un agent (agente). QueueDock ya da a cada mensaje pendiente una fila direccionable, y el transcript (transcripción) durable ya renderiza los eventos de steer consumidos como burbujas de estilo usuario, pero Web no tenía ni una acción que conectara esas dos superficies ni un gesto directo en el composer para elegir el steering (direccionamiento) del turno actual.

Implementar la acción de fila como un borrado del lado del cliente seguido de `session.prompt(mode: 'steer')` partiría una única intención del usuario en dos RPC. El driver podría ganar el reclamo entre ambas, el steer podría fallar tras el borrado, o la alternativa existente de mejor esfuerzo `agent.steer()` podría añadir silenciosamente un nuevo elemento de la cola (Queue) tras eliminar la ocurrencia original. Por tanto, una acción de envío inmediato debe distinguir el steering del turno actual de la promoción desde la cola y conservar la fila original cuando el steering ya no sea posible.

## Decisión

### Contrato de producto

Cada fila de QueueDock de sesión ordinaria sin edición expone la acción de flecha hacia arriba como "插话发送". La acción solo está habilitada mientras la sesión informa de un agent en ejecución; los mensajes de contenido mixto siguen siendo elegibles porque el steering reenvía el `UserMessage` inmutable completo, no la proyección de texto de la fila. Un subagente direccionado mantiene su proyección de la cola de solo lectura porque su transporte de continuación no expone la mutación de la cola.

Activar la acción solicita steering estricto del turno actual para ese `InboxItemId` exacto. El éxito elimina la fila de la cola a través de la instantánea autoritativa del Host e inmediatamente proyecta el mismo steering pendiente tras la fila de estado en ejecución `Deep diving...`; esa burbuja ofrece Copy pero no Fork porque el mensaje aún no tiene una secuencia de eventos durable. Cuando AgentLoop la drena, el evento durable existente `user/message` toma el relevo de la misma burbuja de estilo usuario y restaura su reloj, Copy y Fork sin una vía de presentación durable separada.

El bit de en ejecución es solo una pista de interacción. El valor `acceptsNextStep` de AgentLoop es autoritativo en el límite de mutación síncrona. Si esa ventana se ha cerrado, la operación deja la ocurrencia de la cola sin cambios y devuelve un error tipado `steer-unavailable`, tras lo cual la ocurrencia despertadora original continúa por la cola. Si el driver ya reclamó la ocurrencia, devuelve el error existente `queue-item-not-found` y la entrega como turno independiente ya está en marcha. La UI trata ambas carreras como entrega por la cola convergente sin aviso de fallo; los errores de transporte y desconocidos siguen aflorando.

El composer usa un contrato separado de mejor esfuerzo para la entrada recién tecleada. Mientras la sesión direccionada está inactiva, Enter y Cmd/Ctrl+Enter realizan ambos un envío normal a la cola. Mientras una sesión principal está en ejecución, una preferencia de General Settings asigna Enter solo a Queue (el valor por defecto) o a Steer, y Cmd/Ctrl+Enter realiza el otro comportamiento; Shift+Enter inserta un salto de línea. Un subagente direccionado mantiene ambos gestos en su transporte de continuación solo-Queue. El documento de ajustes del Host persiste la preferencia entre los orígenes Web que comparten un mismo home DSH, y afecta solo al par de gestos de estado ocupado con capacidad de steer. Si un Steer directo del composer falla la ventana actual de next-step, AgentLoop lo admite automáticamente como el siguiente turno despertador de la cola y Web no informa de un fallo.

### Límite de Agent y ciclo de vida

`InboxAction` gana una operación respaldada por Consumer, `{ kind: 'steer' }`, junto a edit y remove. `Agent.updateInbox()` la maneja solo tras localizar la ocurrencia en cola y demostrar `acceptsNextStep`; nunca delega en el alias de mejor esfuerzo `agent.steer()`.

Una acción aplicada termina la ocurrencia en cola y acepta el mismo `UserMessage` inmutable como una nueva ocurrencia de steering. La ocurrencia de steering recibe un nuevo `InboxItemId` y un `placement: 'steering'` veraz, mientras el mensaje conserva su `MessageId`, contenido, fuente y cualquier controlador de entrega `SteeringReceipt` pendiente. AgentLoop instala la nueva entrada del outbox antes de publicar los eventos de ciclo de vida, y luego emite su encolado antes del descarte de la ocurrencia antigua, de modo que la cancelación reentrante no pueda observar ni retirar un elemento no anunciado. Por tanto, el invariante existente de conservación de la inbox sigue exigiendo un encolado y un dequeue o descarte terminal por ocurrencia.

La acción no ejecuta `agent/prompt-submit`: elegir steering cambia deliberadamente la entrega de un turno admitido de forma independiente a entrada de next-step del turno actual. No cancela el trabajo en curso ni reordena el resto de la cola.

### Límite de Host y cliente

`session.updateQueue` transporta la acción `steer` y mapea los dos resultados negativos a errores RPC tipados. La conversión es una única operación síncrona de Agent; el Host nunca la reconstruye combinando llamadas de remove y prompt.

El `queuedMirror` existente del Host sigue siendo la única autoridad transitoria de la inbox. Su instantánea `session/queue` transporta cada ocurrencia viva con `placement: 'queued' | 'steering'`: QueueDock renderiza solo las filas en cola, mientras ChatView renderiza el steering pendiente al final de la conversación tras la fila de estado en ejecución `Deep diving...`, con Copy pero sin Fork, edit ni delete. La reconexión reproduce la misma instantánea, así que esta visibilidad no requiere optimismo del cliente ni un segundo registro.

Cuando AgentLoop reclama el steering pendiente, emite `agent/inbox/dequeue` inmediatamente antes de añadir sincrónicamente el `user/message` durable. El Host retira esa fila de steering en la siguiente microtarea, permitiendo que el evento de sesión durable entre primero en el stream mux lineal. En el evento en vivo aceptado, la Session del cliente retira la primera ocurrencia de steering actual coincidente antes de publicar su instantánea; la reproducción del historial no consume una ocurrencia posterior que reutilizara el mismo `MessageId`. Por tanto, ChatView renderiza una sola autoridad a la vez sin escanear el historial durable, y la proyección durable restaura el reloj, Copy y Fork contra su tiempo y secuencia de evento registrados. Un fallo de append retira igualmente la fila reclamada.

El contrato existente `session.prompt(mode: 'steer')` sigue siendo de mejor esfuerzo para la entrada nueva de la sesión principal: fuera de la ventana de next-step se convierte en un seguimiento despertador. El composer transporta un modo explícito `queue | steer` a través de la adjudicación de slash y la serialización de referencias antes de llamar a ese contrato. Una política de envío del navegador es dueña de la preferencia en vivo de Enter en estado ocupado, mientras que el servicio de ajustes del Host es dueño de la durabilidad; la política resuelve Enter simple frente a Enter acelerado como gestos complementarios solo para sesiones con capacidad de steer, y la fila de Settings y el InputBar la comparten sin duplicar el almacenamiento ni la autoridad de la ventana de entrega. Solo la acción de fila de la cola es estricta, porque cualquiera de los dos resultados negativos converge a través de la ocurrencia original de la cola.

### Verificación

La cobertura del contrato de AgentLoop mantiene abierta la admisión de prompts, convierte una ocurrencia en cola exacta y demuestra que la ocurrencia de steering de reemplazo conserva el valor del mensaje y el recibo de entrega, drena como `user/message` y nunca inicia su antiguo turno independiente. También fija la retención de ventana no disponible, el rechazo de dirección reclamada y la conservación del ciclo de vida ante cancelación reentrante.

Los tests de schema y proxy del Host cubren la nueva acción, ambos errores tipados, las instantáneas conscientes del placement y la reproducción en reconexión, además del orden durable-antes-de-retirada. Los tests de cliente cubren la convergencia silenciosa de ambas carreras semánticas, el reporte genuino de errores, las filas de subagente de solo lectura y los gestos de subagente solo-Queue. Los tests de runtime y de ChatView cubren el handoff pendiente-a-durable consciente de la ocurrencia, incluidos los valores repetidos de `MessageId`, mientras las instantáneas ARIA de Web cubren el steering pendiente tras la fila de estado en ejecución con solo Copy y el nodo durable con reloj, Copy y Fork.

El escenario keyless de steering Web pone en cola un mensaje a través del composer real mientras se transmite la primera respuesta, activa la flecha de la fila y luego usa `ask_user_question` como barrera estable de steering pendiente. Demuestra que la burbuja pendiente respaldada por el Host aparece antes de la admisión, hace handoff a una interjección durable tras la respuesta y afecta a la siguiente solicitud del modelo. Los escenarios ensamblados del composer demuestran que Cmd+Enter en modo por defecto llega a la misma vía pendiente y durable sin crear una fila de la cola, mientras que Cmd+Enter en modo Steer crea una fila de la cola en su lugar. La cobertura de Settings y de la política de envío fija el valor por defecto, la persistencia, el alcance solo-ocupado y el mapeo de gestos complementarios; los escenarios de edit/delete de la cola siguen demostrando que esas acciones no cambian.

## Alternativas consideradas

**Borrar la fila y luego llamar a `session.prompt(mode: 'steer')` desde Web.** Rechazado porque dos RPC no pueden hacer atómicos el borrado y el steering; los fallos y las carreras de reclamo del driver pueden perder o duplicar el mensaje del usuario.

**Restaurar la promoción desde la cola bajo la flecha hacia arriba.** Rechazado porque mover un elemento al frente crea igualmente un turno admitido independiente. El control promete steering del turno actual, no prioridad dentro de la cola.

**Usar el comportamiento existente de mejor esfuerzo `agent.steer()` para la fila de la cola.** Rechazado para esa acción porque una ventana de next-step cerrada crearía una nueva ocurrencia en cola, posiblemente en una posición e identidad distintas. La negativa estricta conserva la ocurrencia original para que la UI pueda tratarla como la misma entrega por la cola aceptada. La entrada recién tecleada en el composer no tiene una ocurrencia en la cola que conservar, así que usa deliberadamente el comportamiento de mejor esfuerzo.

**Cambiar `agent.steer()` para que sea estricto con todos los llamadores.** Rechazado porque los llamadores TUI y de plugins usan su alternativa segura de seguimiento para la entrada recién enviada. Una fila en cola tiene estado recuperable del que esos llamadores carecen.

**Conservar el mismo `InboxItemId` al cambiar el placement.** Rechazado porque `InboxItemId` identifica una aceptación FIFO y `placement` registra la entrega resuelta de esa aceptación. Terminar una ocurrencia en cola y aceptar una ocurrencia de steering mantiene veraces los hechos del ciclo de vida y deja el invariante de conservación sin cambios.

**Añadir una proyección dedicada de steering pendiente y un store de cliente.** Rechazado porque las ocurrencias en cola y de steering ya comparten un ciclo de vida de inbox de Agent y un mirror del Host. Una segunda proyección duplicaría el estado de reconexión y la autoridad de ordenación; una etiqueta de placement permite que cada superficie del cliente seleccione sus filas sin ampliar la semántica de mutación de la cola.

**Cancelar el turno activo y ejecutar el elemento de la cola seleccionado.** Rechazado porque destruye trabajo no relacionado en vuelo y arranca un turno nuevo en lugar de dirigir el actual.

## Consecuencias

`session/queue` describe una instantánea transitoria de la inbox consciente del placement en lugar de una lista solo-Queue, así que cada consumer debe filtrar por placement. El steering pendiente sobrevive a la reconexión y aparece de inmediato, pero sigue siendo no durable hasta que el `user/message` durable se confirma. El bit de en ejecución también puede seguir siendo verdadero brevemente después de que la ventana estricta de next-step se cierre, así que una acción habilitada puede devolver internamente `steer-unavailable` mientras el producto continúa por la cola sin informar de un fallo.

La acción explícita cambia la entrega de un turno admitido independientemente a steering del turno actual, así que los plugins de admisión de prompts no procesan el mensaje convertido. La publicación de ciclo de vida encolado-antes-de-descarte sigue siendo necesaria para la seguridad ante cancelación reentrante; la cobertura de regresión enfocada protege ese orden.
