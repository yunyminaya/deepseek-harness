# Agent Note: Mantener privado el enrutamiento del agent

Status: implemented

[English](2026-07-30-private-agent-send.md) | Español

## Problema

El método público `Agent.send()` exponía la matriz de enrutamiento del bucle concreto aunque los llamadores de producción usan solo las operaciones semánticas `followup()`, `steer()` e `inject()`. Su cuarta combinación, `next-turn` con `wakeup: false`, no tenía consumidor más allá de las pruebas. Mantener pública esa capacidad latente exigía además que las implementaciones alternativas de `Agent` y los fakes de prueba aceptaran política de enrutamiento a nivel de implementación.

## Decisión

`Agent` expone `followup()`, `steer()` e `inject()` como su contrato de entrega completo. `ReactLoopAgent` conserva un helper privado `send()` que comparte la mecánica de enrutamiento entre esos métodos, mientras que `SendTarget` y `SendOptions` dejan de exportarse desde `dsh-agent`.

La interfaz pública no puede poner en cola un turno sin despertar al driver. Un follow-up siempre solicita ejecución, el steering solicita el paso más cercano y la inyección aporta contexto orientado al modelo sin solicitar ejecución. Esto sustituye parcialmente la porción de superficie pública de la [decisión de entrega unificada](../architecture/2026-07-22-unified-send-and-coalesced-user-messages.es.md) conservando su enrutamiento interno y la representación unificada `user/message`.

## Alternativas consideradas

**Mantener pública la matriz de enrutamiento.** Esto conserva la combinación sin uso de cola silenciosa, pero expone mecanismo en lugar de intención del llamador y se la impone a cada driver alternativo.

**Añadir un método público de cola silenciosa.** Un método con nombre sería más claro que banderas de enrutamiento crudas, pero ningún flujo de producción necesita hoy trabajo que permanezca aparcado hasta que una entrega no relacionada lo despierte.

## Consecuencias

Los plugins eligen entre tres operaciones semánticas en lugar de construir opciones de enrutamiento. Los drivers alternativos y los fakes estructurales de prueba implementan un contrato más pequeño, y el catálogo de la API de Cordis ya no anuncia `send`, `SendTarget` ni `SendOptions`.

La capacidad de cola silenciosa eliminada solo puede volver con un consumidor con nombre y semántica de ciclo de vida explícita. `cancel({ keepInbox: true })` sigue preservando el trabajo ya pendiente a través de las rutas de entrega soportadas.
