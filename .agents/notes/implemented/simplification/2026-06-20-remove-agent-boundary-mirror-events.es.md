# Agent Note: Dejar de reflejar los límites durables como eventos de agente

Status: implemented

[English](2026-06-20-remove-agent-boundary-mirror-events.md) | Español

## Problema

El loop registra el transcript canónico en `SessionEvent` y además emitía un conjunto paralelo de eventos espejo de límites `agent/*` en vivo: `agent/turn-start`, `agent/turn-end`, `agent/step-start` y `agent/step-end`. Los espejos obligaban a los consumidores a elegir entre dos fuentes de verdad para el MISMO hecho durable. ACP ya había elegido el registro de sesión para la liquidación de prompts y la salida confirmada porque es el único registro durable y reproducible; consumir un espejo en vivo exigiría reconciliar su temporización con el límite ya almacenado en ese registro. La UI stdio era el único consumidor de producción que aún renderizaba los límites de turno a partir de los eventos espejo; ya renderizaba llamadas y resultados de herramientas desde `session/event`.

Esta duplicación no es gratuita. Cada cambio del ciclo de vida tenía que actualizar el evento de sesión, el evento espejo, la documentación, los invariantes, las pruebas y las expectativas de instantáneas. Los eventos de límite duplicados además hacían sutil el orden de los fallos: un turno puede cerrarse durablemente antes de que un listener en vivo de `agent/turn-end` se ejecute, así que un fallo del listener posterior al límite no deja ninguna posición válida en el registro y debe notificarse fuera de banda.

## Decisión

Convertir `session/event` en la única transmisión en vivo de límites/transcript. Los consumidores que renderizan turnos, llamadas de herramientas, resultados de herramientas, mensajes de asistente y límites durables se suscriben a `session/event` y derivan su UI del mismo vocabulario de eventos que usa la persistencia.

Los cuatro espejos de límites durables — `agent/turn-start`, `agent/turn-end`, `agent/step-start`, `agent/step-end` — se eliminan de la taxonomía de eventos de agente. Una UI que quiera el handle del agente en un límite conserva el objeto destino en vivo de `agent/created`/`agent/disposed` y compara su sesión directamente; `dsh-ui-stdio` usa esto para etiquetar el encabezado `[main turn N]` del agente propiedad de la app mientras que otras sesiones renderizan su id durable. El registro canónico sigue siendo el registro de sesión basado en eventos.

Los espejos de paso (que no tenían ningún consumidor) se eliminaron primero, en el [Agent Note de semántica de dominio de eventos](../architecture/2026-06-30-event-domain-semantics.es.md); ese Agent Note CONSERVÓ los espejos de turno con la justificación declarada de que la UI stdio necesitaba el handle `Agent` en el límite del turno. Esta decisión termina el trabajo: `dsh-ui-stdio` es un REPL de prueba desechable cuyo renderizado puede cambiar libremente, así que «ui-stdio lo necesita» no es razón para conservar un espejo — lee `session/event` y conserva solo su objeto destino en vivo.

## Alcance: qué se elimina y qué no

Eliminados (espejos de límites durables — el registro de sesión es autoritativo para cada uno): `agent/turn-start`, `agent/turn-end`, `agent/step-start`, `agent/step-end`.

CONSERVADOS — NO son espejos de límites durables, así que quedan fuera del alcance de esta decisión:

- `agent/steering` — no es un límite, así que queda fuera del alcance de ESTA decisión. Refleja el registro de control durable `steering/message` en lugar de un límite, y fue eliminado por su propio seguimiento: [Eliminar la emisión del espejo `agent/steering`](../../archived/simplification/2026-07-04-remove-agent-steering-mirror.md).
- `agent/stream-chunk` — la transmisión de tokens en vivo. Fuera del alcance de ESTA decisión (un espejo de `assistant/chunk` durable, no un límite), fue eliminado por su propio seguimiento: [Dejar de reflejar la transmisión de tokens como evento de agente](../../archived/simplification/2026-07-02-remove-stream-chunk-mirror.md).
- `agent/created`, `agent/disposed`, `agent/status`, `agent/error`, `agent/queued` — eventos de ciclo de vida/control que no son datos de transcript. `agent/queued` en particular es un acuse de recibo de la bandeja de entrada que se dispara antes de que exista cualquier evento durable (el trabajo encolado cancelado puede nunca entrar en el registro), así que es deliberadamente solo-en-vivo.

## Alternativas consideradas

- **Incluir `agent/steering` en la eliminación** — la forma de la propuesta original; se acotó por ser ampliación de alcance: refleja el registro de control durable `steering/message`, no un límite, y fue eliminado por [su propia decisión posterior](../../archived/simplification/2026-07-04-remove-agent-steering-mirror.md) (y `agent/stream-chunk`, por el [Agent Note del espejo stream-chunk](../../archived/simplification/2026-07-02-remove-stream-chunk-mirror.md)).
- **Conservar los espejos de turno para la UI stdio** — la postura original del [Agent Note de semántica de dominio de eventos](../architecture/2026-06-30-event-domain-semantics.es.md); rechazada aquí porque `dsh-ui-stdio` es un REPL de prueba desechable, no un consumidor que soporta carga, y renderiza los límites desde `session/event` más su objeto destino en vivo.

## Consecuencias

Un plugin ya no puede observar los límites de turno/paso desde un evento conveniente que prioriza `Agent`. Se suscribe a `session/event` y, si necesita el objeto en vivo, resuelve el id compartido mediante `ctx.agents` o conserva el objeto que ya posee. Es un intercambio aceptable: los consumidores de límites no deberían depender de un segundo flujo de eventos que puede desviarse del registro durable.
