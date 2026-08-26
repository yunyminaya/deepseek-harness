# Agent Note: Los eventos con ámbito de agent despachan un único objeto payload

Status: implemented

[English](2026-08-06-agent-event-payload-objects.md) | Español

## Problema

Los eventos con ámbito de agent (agente) recibían históricamente argumentos posicionales: un sujeto `agent` inicial, campos específicos del evento y un `next` final para los eventos waterfall (cascada de eventos)/seriales. Añadir un campo o retirar un tipo de contexto (como con `PreStepContext` y `RequestFailureContext`) obligaba a reescribir todos los listeners y emisores de todos los paquetes, y el contrato seguía repartido por la lista de parámetros en lugar de concentrarse en un payload con nombre.

## Decisión

Cada evento con ámbito de agent recibe exactamente un objeto payload como primer argumento. El payload siempre lleva el sujeto (`agent`), los campos del evento y la `signal` de cancelación cuando el evento la tiene; `next` sigue siendo el último argumento de los eventos waterfall/seriales. Los eventos afectados son los doce eventos `agent/*`, `agent-loop/config-start-failed` (el único sin sujeto) y `goal/changed`.

`PreStepContext` y `RequestFailureContext` se retiran; sus campos viven directamente en los payloads de `agent/pre-step` y `agent/request-error`.

El despacho está fusionado: `agentEvents(ctx, agent)` (y el `emitAgentEvent` de un solo uso) inyecta el sujeto para que la clave del portador de ámbito y el `agent` del payload no puedan divergir, y el sujeto inyectado gana incluso sobre un payload estructuralmente aceptable que casualmente lleve un campo `agent`. `ReactLoopAgent` construye su dispatcher una sola vez en el constructor y enruta por él todos los emit, seriales y waterfalls, de modo que los despachos en la ruta caliente no asignan nada.

## Alternativas consideradas

**Mantener las firmas posicionales.** Añadir un campo o retirar un tipo de contexto seguiría obligando a reescribir todos los listeners y emisores, y el contrato seguiría repartido por la lista de parámetros en lugar de concentrarse en un payload con nombre.

**Construir el sujeto a mano en cada punto de despacho.** El diseño intermedio del loop llamaba a `ctx.waterfall(this.carrier, …)` con un payload `{ agent: this, … }` construido manualmente; evitaba la asignación por despacho, pero duplicaba la inyección del sujeto y dejaba que la clave de ámbito y el sujeto del payload divergieran. El dispatcher fusionado es el único punto de inyección para todos los modos de despacho.

## Consecuencias

Las firmas de los listeners nombran el payload completo una sola vez, de modo que ampliar un payload o retirar un tipo de contexto es un cambio de una única forma en todos los listeners y emisores. El acoplamiento sujeto/ámbito lo impone el dispatcher en todos los modos de despacho, y las rutas calientes del loop siguen sin asignar nada.
