# Agent Note: Separar la inyección de contexto de la ejecución del turno

Status: implemented

[English](2026-07-24-separate-context-injection-from-turn-execution.md) | Español

## Problema

La API del agent (agente) representaba la entrada suplementaria orientada al modelo de tres maneras solapadas: los llamadores adjuntaban `HookContext[]` mediante `SendOptions.contexts`, los hooks de intercepción y de herramientas devolvían `additionalContexts`, y los plugins llamaban a `agent.inject()`. Estas vías acababan escribiendo contexto en el mismo historial del modelo, pero llevaban reglas distintas de colocación, metadatos, admisión, cola y ciclo de vida del turno.

La fijación atómica a un mensaje del inbox obligaba al agent loop a conservar el contexto a través de la admisión de prompts, la conversión a steering (direccionamiento), la cancelación y el descarte final. La colocación `prompt-prefix` combinaba después el contexto y el prompt directo en un único evento, lo que exigía un sobre oculto al modelo para que los consumidores de transcript (transcripción) pudieran recuperar lo que el usuario escribió realmente. El resultado hacía que las entradas del outbox, la proyección de la sesión y la reproducción en la UI fueran responsables de una distinción que pertenece al productor.

El `inject()` inactivo exponía un segundo desajuste. La inyección no solicitaba ejecución del modelo y, sin embargo, la implementación abría y cerraba un turno de `injection` de cero pasos solo para satisfacer el invariante de encierro del turno y obtener un punto de control de durabilidad. Un turno significaba, pues, a veces «ejecutar el agent loop» y a veces «persistir contexto sin ejecutarlo».

`HookContext` también nombraba a su productor en lugar de su rol. El valor podía venir de un plugin nativo, de un hook bridge, de la admisión de prompts o del postprocesado de herramientas; su significado estable era contexto adicional orientado al modelo cuya fuente nombraba al productor.

## Decisión

`inject()` es la única operación orientada al llamador para la entrada suplementaria orientada al modelo, y un turno significa una ejecución del bucle del modelo.

Un llamador que posee contexto entrega un `UserMessage` identificado y congelado mediante `inject()` y envía el mensaje directo de forma independiente con `followup()` o `steer()`.

Un pre-step entrante devuelve el lote completo `PreStepDecision.messages` para la petición que se está finalizando. Los puntos de extensión de herramientas siguen devolviendo `additionalContexts`, que entran en el inbox del siguiente paso solo después de los resultados de la herramienta correspondiente. Estos valores son salidas de puntos de extensión, no adjuntos capturados de un elemento del inbox del llamador.

Cada contexto adicional es un `UserMessage` independiente cuyo `source` nombra a su productor y lleva campos específicos del productor. La inserción en el inbox es durable de inmediato; la admisión registra después el mismo valor como `user/message`. No existe `context/message`, ni colocación prompt-prefix, ni delimitador estable de petición, ni sobre de prompt. Los consumidores de transcript y de UI distinguen los mensajes directos del usuario del contexto inyectado mediante `source`.

## Ciclo de vida de la inyección

`inject()` inserta siempre el contexto en el inbox `next-step` sin despertar y confirma esa mutación de cola como `agent/inbox/spliced`. Un driver en ejecución la reclama en el límite de pre-step posterior más cercano. Un driver inactivo la deja pendiente hasta que `followup()` o `steer()` suministren trabajo que despierte; la cancelación o el dispose pueden descartarla antes sin borrar el historial durable de la cola.

El agent loop reclama el lote actual de next-step antes de ejecutar `agent/pre-step`, de modo que una inyección que llega después de esa reclamación puede perder la petición que ya se está finalizando. El siguiente límite la reclama en su lugar. Una decisión de enter añade sus mensajes devueltos dentro del turno propietario antes de que la petición los consuma. El contexto producido durante un lote de llamadas de herramienta del asistente aparece, por tanto, después de los resultados ordenados completos de ese lote.

Si pre-step rechaza o lanza un error, su contexto inyectado reclamado, su steering y el prompt en cola permanecen retirados y no se añade ningún lote devuelto. Los mensajes insertados después de esa reclamación atómica no se ven afectados y siguen pendientes.

El agent loop añade eventos `user/message` inyectados solo a partir de lotes entrados dentro de un turno. Los eventos de ejecución del core, el steering, la salida del asistente y las herramientas permanecen encerrados en el turno; las relaciones de eventos extensibles por merge pertenecen al plugin que las declara, no a un valor por defecto del core.

## Semántica de extensión y de llamador

El `PreStepDecision.messages` de la rama enter es el lote completo para el paso propuesto. Un listener de waterfall (cascada de eventos) que delega con `next()` conserva los mensajes posteriores salvo que los sustituya deliberadamente; las adiciones siguen el orden de retorno natural del waterfall. Los `additionalContexts` de resultados de herramientas conservan el orden FIFO y el source de cada mensaje.

La inyección impulsada por el llamador y el contexto del paso actual usan deliberadamente tiempos distintos. `inject()` se incorpora al siguiente pre-step disponible y no puede prometer que una petición ya en finalización lo consuma. Un listener que deba afectar a esa petición exacta devuelve el contexto en `PreStepDecision.messages`; un rechazo o fallo posterior impide entonces que se materialice.

Las referencias entre sesiones usan esa composición de dominios: el TUI prepara la instantánea, la devuelve desde el pre-step del mensaje directo inactivo junto a ese mensaje, o la inyecta antes de despertar el steering durante un turno en ejecución. El registro objetivo contiene dos mensajes simples, de modo que una mutación posterior de la fuente no puede cambiar la reproducción y los consumidores de transcript no necesitan un sobre de prompt. Esto sustituye al mecanismo de adjunto de la [decisión de referencias entre sesiones](../feature/2026-07-21-cross-session-references.es.md) conservando sus reglas de instantánea y de límite de confianza.

Esta decisión conserva la decisión de encuadre propiedad del llamador de [contenido inyectado sin envolver](../simplification/2026-07-20-unwrap-injected-content-envelopes.es.md) y la regla de un turno por envío de [un envío, un turno](../simplification/2026-07-17-one-send-one-turn.es.md). La posterior [decisión de eventos solo-log independientes](../simplification/2026-07-28-remove-synthetic-log-only-turns.es.md) aplica el mismo significado de solo-ejecución a los registros propiedad del plugin.

## Alternativas consideradas

**Mantener `SendOptions.contexts` como adjunto atómico.** Esto preserva la entrega de todo-o-nada cuando la admisión de prompts bloquea, pero mantiene el contexto dentro del estado del ciclo de vida del inbox y exige que cada transición de cola y cada evento de observación lo transporte. La API genérica del agent no debería codificar una transacción de dominio que la mayoría de los llamadores pueden expresar como inyección de contexto seguida de entrega de mensaje.

**Mantener un evento de sesión `context/message` distinto.** La entrada de modelo con rol de usuario tendría de nuevo dos tipos de evento con proyección idéntica. `user/message.source` ya lleva la distinción que necesitan las políticas, el transcript y los consumidores de reproducción.

**Mantener turnos de un solo uso para la inyección inactiva.** La inserción durable en el inbox ya registra el contexto inactivo sin abrir un turno. Un turno sintético haría que los contadores de turnos y los observadores informaran de trabajo que nunca ejecutó el modelo; el contexto que no despierta permanece pendiente hasta que un trabajo que despierte real aporte una petición.

**Mantener `prompt-prefix` como colocación opcional.** El horneado del prefijo puede hacer que el contexto y la petición aparezcan en un único mensaje del provider, pero introduce una segunda representación del prompt directo y reparte el manejo de la colocación entre admisión, steering, registro, reproducción y código de UI. Los productores que requieran encuadre textual pueden incluirlo en su propio contenido de contexto.

**Dejar que los hooks de prompt llamen a `inject()` en lugar de devolver mensajes.** Una inyección podría perder la petición cuyo prompt ya se está finalizando y escaparía a un bloqueo posterior de esa decisión. Devolver el lote completo de mensajes mantiene el contexto de la petición actual bajo la autoridad del waterfall.

## Verificación

- Las entradas de entrega y los registros del inbox de steering no contienen contextos adjuntos; `agent/inbox/inserted` informa solo del mensaje insertado, mientras que el splice durable conserva su lista objetivo.
- `UserMessage` es la forma compartida, identificada y congelada en intercepción de prompts, ejecución de herramientas, hook bridges, guardas y productores de contexto.
- La colocación prompt-prefix, los sobres de prompt y `context/message` están ausentes de los tipos públicos, los eventos durables, la proyección y la reproducción de UI.
- El `inject()` inactivo añade de inmediato una inserción durable en el inbox, pero ningún `user/message` visible al modelo; una entrega que despierte posterior puede iniciar el procesamiento de pre-step.
- La inyección durante un turno activo se reclama en el límite de pre-step posterior más cercano, después de los lotes completos de resultados de herramienta y antes de la petición que la consume.
- Un pre-step rechazado o fallido descarta su lote reclamado; la entrada insertada después de la reclamación permanece pendiente.
- La cobertura de unidades, persistencia/reanudación, invariantes y TUI fijan el orden de eventos, la propiedad de la reclamación y la reproducción durable.

## Consecuencias

- La inyección inactiva no es visible al modelo hasta que un pre-step posterior la hace entrar y puede ser descartada por cancelación o dispose, mientras su ciclo de vida durable en el inbox queda registrado.
- Los mensajes consecutivos con rol de usuario sustituyen a un único mensaje de prompt horneado; los adaptadores de provider conservan ese orden.
- El contexto exacto de la petición actual debe devolverse desde `agent/pre-step`; la inyección ordinaria solo ofrece entrega en el límite posterior más cercano.
- El contrato público de entrega y los registros del inbox siguen siendo pequeños: sin adjunto de contexto, sin metadatos de colocación de contexto, sin sobre de prompt ni tipo de evento durable duplicado.
