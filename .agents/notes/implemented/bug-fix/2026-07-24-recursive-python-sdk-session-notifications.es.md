# Agent Note: Notificaciones recursivas de sesión del SDK de Python

Status: implemented

[English](2026-07-24-recursive-python-sdk-session-notifications.md) | Español

## Problema

El SDK de Python filtraba las notificaciones de turno comparando cada carga útil directamente con el id de la sesión raíz. Eso admitía el ciclo de vida de un hijo directo porque su id padre nombraba la raíz, pero rechazaba el ciclo de vida de un nieto y cada `session.event` descendiente. El servidor JSON-RPC seguía emitiendo esas notificaciones, así que se acumulaban en la cola global de bajo nivel mientras los consumidores de alto nivel perdían las relaciones de trayectoria anidada y los estados de finalización.

## Decisión

`HarnessClient` registra cada arista hijo-a-padre válida de `subagent.started` antes de despachar la notificación. Un `subagent.finished` posterior se enruta por su id padre inmutable pero nunca reescribe la ascendencia actual, de modo que una ejecución antigua que se asienta después de que su id hijo se haya reutilizado no puede desplazar a la sesión de reemplazo. Las demás notificaciones de sesión resuelven su id de sesión recorriendo ese grafo de ascendencia de la vida del cliente hasta la raíz solicitada. El grafo sobrevive a suscripciones sucesivas, así que un descendiente que sobrevive a un `Session.run()` sigue siendo atribuible cuando emite durante un turno posterior, y se reinicia cuando el cliente arranca un nuevo proceso del runtime.

`Session.run()` entrega el flujo completo de notificaciones del árbol de sesiones descubierto a través de `TurnResult.notifications` y `on_notification`. Solo las notificaciones `session.event` cuyo `sessionId` coincide con la raíz solicitada entran en `TurnResult.events` o en la reconstrucción de la respuesta final. Los eventos descendientes son por tanto observables sin permitir que una respuesta hija reemplace la respuesta raíz.

## Alternativas consideradas

**Añadir un id de sesión raíz a cada notificación JSON-RPC.** El servidor ya proporciona las aristas exactas de padre inmediato, y duplicar la ascendencia transitiva en el protocolo haría a cada productor responsable del estado de suscripción del cliente.

**Limitar los subagentes a un nivel.** Un despliegue puede fijar `maxDepth: 1`, pero cambiar el SDK para depender de esa política informaría mal, en silencio, composiciones recursivas válidas.

**Suscribirse solo a las notificaciones de ciclo de vida descendientes.** Repararía el informe de relaciones y de finalización, pero los eventos de sesión descendientes seguirían acumulándose en la cola global y los callbacks expondrían un árbol incompleto.

**Exponer e indexar cada id de ejecución de subagente en el protocolo JSON-RPC.** La identidad exacta de ejecución es útil cuando un cliente debe correlacionar dos desenlaces concurrentes del mismo hijo, pero el enrutamiento por árbol de sesión ya tiene la arista de inicio autoritativa y el padre inmutable de cada notificación terminal. Ampliar el protocolo es innecesario para esta decisión de propiedad.

## Consecuencias

Los consumidores de alto nivel reciben el ciclo de vida anidado y las notificaciones de sesión en orden de protocolo mientras los resultados de turno raíz conservan su semántica de respuesta previa. El cliente retiene una entrada padre actual por cada hijo observado hasta que el runtime se reinicia; la búsqueda de ascendencia es segura ante ciclos, y las notificaciones de sesión no relacionadas siguen disponibles a través de la cola global. Las pruebas sin clave de Python cubren la delegación de dos niveles, el aislamiento de la respuesta raíz, la ausencia de acumulación en la cola de notificaciones del árbol, la reutilización de ascendencia entre suscripciones y los ids hijo reutilizados cuyas ejecuciones antiguas se asientan fuera de orden.
