# Agent Note: Conservar una única primitiva pública de parada

Status: implemented

[English](2026-06-20-public-agent-stop-api.md) | Español

## Problema

El handle público `Agent` exponía dos formas solapadas de detener trabajo en curso: `abort()` solo de paso y `cancel()` consciente de la cola. La primera preservaba la entrada encolada, mientras que la segunda originalmente solo exponía su amplio comportamiento por defecto, que limpia el trabajo encolado y de steering mientras aborta el turno activo. `cancel(cause, { keepInbox: true })` ahora cubre la política de parada Web de producción sin exponer el titular privado del turno; ACP conserva la cancelación amplia, mientras que los propietarios del ciclo de vida destruyen los agentes mediante `AgentHandle.dispose()`. Ningún llamador de producción necesita un abort solo de paso sin más.

La distinción de comportamiento es real, pero ningún código publicado necesita un verbo más estrecho separado. AgentLoop posee un único titular de cancelación privado para todo el turno. `cancel(cause, options?)` lleva una causa `user` o `parent` explícita y tipada; su comportamiento por defecto amplio descarta la entrada pendiente, mientras que `keepInbox` preserva el trabajo pendiente para turnos posteriores. La destrucción sigue siendo una interrupción separada del ciclo de vida. El contrato completo de propiedad y propagación vive en la [nota de cancelación explícita de turno](../architecture/2026-07-16-explicit-turn-cancellation.es.md).

La superficie adicional hacía que el loop llevara un verbo público que era en su mayor parte un interno de teardown. Un `cancel()` con opciones expresa la política del llamador sin exponer una segunda operación con forma de titular.

## Decisión

`cancel()` es la única primitiva pública de *parada* en `Agent`. Los propietarios del ciclo de vida usan `AgentHandle.dispose()` para detener y desregistrar un agente; los no propietarios usan `cancel()` amplio para abandonar el trabajo actual y encolado, o `keepInbox` para abortar el turno activo conservando el trabajo pendiente. La implementación conserva un titular privado de cancelación del turno, pero no forma parte del contrato `Agent` visible para plugins. La [decisión de parada Web](../bug-fix/2026-07-31-web-stop-preserves-queue.es.md) es el consumidor de producción de `keepInbox`.

`whenIdle()` se **conserva** como la primitiva pública de observación de quietud (resuelve una vez que el agente sale de `running`, resuelve de inmediato cuando ya está inactivo, y espera la salida del loop al destruirse). No es un verbo de parada; es cómo un no propietario observa que la parada *se completa* sin destruir el agente. Sus consumidores vivos son ACP y las pruebas de agentes que esperan la estabilización a través de este contrato público (`packages/acp/acp/tests`, `packages/core/agent-loop/tests`); el puente ACP de producción es propietario de sus agentes y los destruye mediante `AgentHandle.dispose()`, así que `packages/acp/acp/src` no tiene ninguna llamada a `whenIdle()`.

El `abort()` público está ausente, y el disposer sigue siendo asíncrono y espera a que el loop se detenga. Las pruebas ejercitan la cancelación a través de la causa tipada pública y las API de señal explícitas, sin llegar al titular.

## Alternativas consideradas

**Eliminar también `whenIdle()`** — la forma de la propuesta original, invertida al validar la premisa contra el código: es una primitiva de quietud que soporta carga, maneja con seguridad las carreras entre espera de settlement y turnos de reemplazo, y empujar a los consumidores hacia transiciones `running`→`idle` observadas a mano es exactamente el camino frágil contra el que advierten los patrones defensivos.

## Verificación

`Agent` no expone ningún `abort()` público mientras `cancel()`, `whenIdle()` y `steer()` permanecen; la cancelación de ACP llama a `cancel()` amplio, la parada Web llama a `cancel(..., { keepInbox: true })`, y el teardown espera la quietud mediante la destrucción del handle. `whenIdle()` resuelve en la quietud para los observadores no propietarios, y las suites cubren la cancelación y la destrucción como las dos vías de parada soportadas.

## Consecuencias

Un plugin puede abortar el turno activo preservando los prompts encolados mediante `keepInbox`, pero no puede abortar solo un paso de modelo/herramienta dejando ese turno en marcha. Un caso de uso solo de paso necesitaría un consumidor con nombre y un contrato más estrecho; exponer el mecanismo privado del loop sigue sin estar justificado.

## Relacionado

Este Agent Note solo elimina el verbo de parada redundante. El steering a mitad de turno sigue siendo una vía de mensajes intencional; la observación de quietud sigue disponible mediante `whenIdle()`. La superficie de entrega resultante es `followup()`, `steer()` e `inject()`; la parada y la observación permanecen con `cancel()` y `whenIdle()`.
