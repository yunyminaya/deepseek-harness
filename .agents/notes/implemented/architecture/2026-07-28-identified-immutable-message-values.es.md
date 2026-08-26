# Agent Note: Crea cada mensaje como un valor inmutable con identidad

Status: implemented

[English](2026-07-28-identified-immutable-message-values.md) | Español

## Problema

El harness tenía varias representaciones con forma de mensaje con reglas de identidad distintas. La entrada del agent solo adquiría un id de correlación de inbox cuando el loop la aceptaba, mientras que los mensajes de usuario duraderos, los mensajes de assistant, los resultados de herramienta y los mensajes de petición de modelo podían carecer de identidad. La admisión de prompt quedaba así entre la creación y la identidad, y el contenido equivalente se copiaba entre eventos en vivo, eventos duraderos y peticiones de modelo sin un único valor que nombrara el mensaje a lo largo de toda su vida.

Eso convertía la identidad en un efecto secundario del enrutado y no en un invariante del mensaje. Los productores no podían referirse a un mensaje antes de llamar al agent, los hooks de prompt recibían contenido y source por separado, y las proyecciones posteriores tenían que reconstruir un mensaje decidiendo a la vez si existía un id. La inmutabilidad también empezaba en límites distintos: algunos inputs los congelaba el loop, otros solo el append de sesión, y la salida de assistant producida por el provider usaba una forma aparte que transportaba provider, modelo y estado de replay.

## Decisión

`@deepseek-ai/dsh-llm` es dueño de un único valor `Message` con `id`, `role`, `content` y `source` obligatorios. `MessageId` es opaco y lo comparten los mensajes de usuario, de assistant y de resultado de herramienta. Un mensaje recibe su id en la creación, antes del enrutado de inbox, del claim, de la reescritura de pre-step, del append duradero o de la proyección de petición. El mismo id sobrevive a todos los límites de representación.

`createMessage(input)` es la frontera canónica de creación genérica por rol. Acuña un `MessageId`, desacopla el role, el content y el source suministrados, y congela en profundidad el valor completo antes de devolverlo. `createUserMessage({ content, source })` fija el rol de usuario para los productores de prompt y de contexto. `createAssistantMessage({ content, source })` fija a la vez el rol de assistant y el tipo de source de modelo, de modo que los productores de salida de modelo suministran solo content más provider, modelo y estado de replay opcional. Todos los helpers de creación excluyen un id de entrada para que los llamadores no puedan presentar por accidente una creación como import. `freezeMessage(message)` es la frontera separada de import o transformación: desacopla y congela en profundidad un mensaje cuya identidad ya existe, sin acuñar un reemplazo.

Los helpers viven en `dsh-llm`, junto al vocabulario base de mensajes, porque sus contratos completos dependen solo de ese vocabulario. `createToolResultMessage()` está con los demás helpers de creación: acopla un id de llamada de herramienta al bloque exacto de resultado de herramienta con rol de usuario y a su source, sin depender del estado de sesión ni de los eventos. `dsh-session` consume mensajes completos en lugar de ser dueño de su construcción.

La interfaz `Agent` acepta un `UserMessage` completo a través de `followup`, `steer` e `inject`. Estas operaciones nunca asignan ni devuelven identidad; congelan un valor importado cuyo id ya posee el llamador. Los claims de inbox y `agent/pre-step` reciben ese mensaje directamente. Una reescritura de contenido crea un reemplazo congelado con el mismo id, mientras que un contexto adicional es un `UserMessage` creado por separado con su propio id.

Los eventos duraderos que producen mensajes almacenan mensajes completos. `user/message` almacena su `UserMessage` directamente; `assistant/message` y `tool/result` envuelven su mensaje especializado por rol junto a hechos locales del evento como posición, uso, fallo o presentación. La derivación de sesión devuelve esos valores congelados en lugar de reconstruir mensajes anónimos. El ensamblaje de assistant crea un mensaje con source de modelo cuando una respuesta se completa, y la ejecución de herramienta crea un mensaje con source de herramienta cuando se confirma un resultado.

Cualquier operación que cambie solo la representación de un mensaje semántico existente conserva su id y devuelve otro valor congelado. Una operación que crea un mensaje semántico nuevo acuña un id nuevo. Las reescrituras de contenido de compactación conservan por tanto la identidad del resultado de herramienta reescrito, mientras que un checkpoint de resumen es un mensaje nuevo.

## Alternativas consideradas

**Mantener los ids opcionales en el mensaje base.** Esto minimizaría la migración de fixtures y permitiría que las formas de provider o de persistencia siguieran siendo anónimas. También conservaría la ambigüedad original: todo Consumer tendría que bifurcar según exista o no la identidad, y ningún tipo demostraría que la admisión, el registro o la proyección la retuvieron.

**Dejar que la entrega del agent asigne el id.** Esto mantiene la identidad acotada a la correlación de inbox, pero convierte la llamada al agent en el punto más temprano en que un productor puede nombrar su propio mensaje. La construcción de prompt, los adjuntos de la UI y la coordinación síncrona de enqueue/descartes necesitan entonces coincidencia de contenido o un token fuera de banda antes de que la entrega devuelva.

**Dejar que cada evento duradero asigne un id nuevo.** Esto da identidad a los mensajes persistidos pero rompe deliberadamente la correlación con el input en vivo y hace que las peticiones replayed parezcan contener mensajes distintos. La identidad pertenece al valor semántico, no a cada envelope que lo transporta.

**Congelar solo en la admisión del agent o de la sesión.** Esto evita un helper de creación pero deja un intervalo mutable identificado en el que el código del llamador puede cambiar el significado asociado a un id. La decisión hace que «tiene un id» y «es una instantánea inmutable» coincidan.

## Consecuencias

Todo productor de mensajes debe elegir explícitamente entre creación e import, y los tests construyen valores completos en lugar de registros parciales de content/source. La generación de UUID se desplaza hacia fuera, al primer punto de creación semántica, de modo que los fixtures deterministas que aportan un id existente usan `freezeMessage()` en lugar de `createMessage()`.

Los eventos de inbox en vivo, los eventos duraderos, el historial derivado y las peticiones de modelo pueden correlacionar un mensaje sin igualdad de contenido ni ids específicos de envelope. La política de inputs pendientes y la limpieza de adjuntos de la UI pueden comparar `MessageId` antes de que exista un turno, mientras que los claims conservan esa identidad dentro del turno abierto. La congelación en profundidad impide que un productor, un hook o un observador cambien el valor una vez establecida la identidad.

La representación compartida elimina la antigua división `UserMessageData`/`AgentMessage` y sitúa provider, modelo y estado de replay opcional en sources de mensaje tipados. Los envelopes de evento siguen siendo dueños de hechos que no son semántica de mensaje, como la posición de turno y de step, el uso de tokens, la identidad de fallo interno de herramienta y los metadatos de presentación.

Los tests unitarios de mensajes y helpers fijan la identidad inmediata, el desacoplamiento, la inmutabilidad profunda y la conservación de un id importado. Los tests del agent loop fijan la identidad a lo largo de la admisión, el ciclo de vida de inbox, el append duradero, la reescritura de contenido y la cancelación; los tests de sesión fijan la derivación congelada y el reemplazo que conserva la identidad.

## Relacionado

- [Enrutado unificado de entrega del agent y contexto inyectado coalescido](2026-07-22-unified-send-and-coalesced-user-messages.es.md) — esta nota supera sus detalles de representación de entrada e id asignado por el agent, conservando su decisión de enrutado.
- [Peticiones reconstruibles](2026-07-05-reconstructable-requests.es.md) — el log de sesión sigue siendo la autoridad para toda entrada visible por el modelo.
