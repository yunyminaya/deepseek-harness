# Agent Note: Eliminar los eventos durables de límite de paso

Status: rejected — `step/end` es la indicación durable de que un paso del modelo terminó, y conservar el par simétrico `step/start` / `step/end` hace que la reparación tras caídas, los invariantes y la inspección del transcript sean más claros que inferir la finalización a partir de eventos adyacentes con ámbito de paso.

[English](2026-06-20-drop-durable-step-boundaries.md) | Español

## Problema

El log de sesión almacena los eventos `step/start` y `step/end` aunque cada evento con ámbito de paso ya lleva `{ turn, step }`: chunks de assistant, mensajes de assistant, llamadas de herramienta, resultados de herramienta, usage y errores. `deriveMessages()` ignora los límites de paso, ACP los ignora para la UI, y los principales consumidores son los invariantes, las pruebas, las salidas esperadas de instantánea y la reparación tras caídas.

El argumento rechazado era que los eventos de límite hacen el log más ceremonial que informativo. En la práctica, `step/end` es información concreta: un lector puede saber si una petición al modelo terminó, falló o está en reparación sin derivar ese estado del siguiente evento. Un `step/start` aislado es igualmente útil para una petición al modelo que comenzó pero no produjo chunks antes de fallar.

## Propuesta

Hacer del turno el único límite durable. Eliminar `step/start` y `step/end` de `SessionEventMap`; conservar el campo numérico `step` en los eventos que necesitan agrupación. El loop incrementa el contador de pasos y registra los eventos con ámbito de paso con ese número, pero ya no añade eventos de límite de apertura/cierre. Los consumidores infieren los grupos de pasos a partir de eventos contiguos que comparten `(turn, step)`.

El plugin de invariantes debería exigir que los eventos con ámbito de paso tengan números de paso positivos válidos dentro de un turno abierto, no que registros de límite separados los rodeen. La reparación tras caídas no debería sintetizar `step/end`; si un turno interrumpido se conserva, la ruta de reparación puede seguir cerrando el turno sin inventar registros de límite de paso.

## Criterios de aceptación

- `SessionEventMap` ya no incluye `step/start` ni `step/end`.
- El loop no tiene ruta de finalización `closeStep()`.
- Las instantáneas de ACP y los fixtures de contrato de persistencia dejan de esperar líneas de límite de paso.
- `deriveMessages()` y el replay derivan el mismo historial de mensajes de los eventos con ámbito de paso.
- Los [docs de taxonomía de eventos](../../../../docs/architecture.es.md) describen los turnos como el límite durable y los pasos como un campo de los registros con ámbito de paso.
- Se refrescan la versión del formato de sesión y los fixtures grabados; los logs almacenados no actuales se rechazan según la política de formato pre-release.

## Qué renunciamos

El log ya no registra «una petición al modelo comenzó pero no produjo ningún evento antes de morir el proceso» como un hecho durable, y ya no tiene un marcador explícito de «este paso se completó». Esa pérdida no es aceptable mientras el log de sesión sea la superficie durable de replay y auditoría.

<!-- agent-note-format: alternatives-not-recorded (pre-format Agent Note) -->
