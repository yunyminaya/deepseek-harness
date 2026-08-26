# Agent Note: Event-sourced sessions with derived message history

[English](2026-06-11-event-sourced-sessions.md) | Español

Status: implemented

## Problem

El MVP requiere traza estricta basada en eventos con sesiones completamente reproducibles (严格的基于事件的trace、logging系统，session完全可回放).

## Decision

Una `Session` es un registro de solo anexión de `SessionEvent`s tipados — la única fuente de verdad. El historial de mensajes del LLM se *deriva* del registro (`deriveMessages()`); los chunks de flujo sin procesar se registran para fidelidad de reproducibilidad a nivel de token mientras el evento `assistant/message` ensamblado es autoritativo para la derivación. Replay/forke = sembrar una nueva sesión con un registro existente.

Los anexiones son síncronos (la ruta caliente nunca bloquea en E/S); `session/event` es una notificación síncrona; los plugins de persistencia almacenan en búfer la escritura asíncrona y drenan en el punto de control `session/flush` esperado disparado al final de cada turno.

Contrato de orden: el bucle reclama los mensajes de bandeja de entrada antes de `agent/pre-step`, abre `step/start` solo después de una decisión de entrada, luego anexa el lote `user/message` devuelto antes de la derivación de solicitudes. La salida del proveedor se ensambla y se anexa como `assistant/message` antes del despacho de herramientas, por lo que el registro durable registra el mensaje exacto que siguen las herramientas. Las pruebas de regresión fijan ese orden.

## Alternatives considered

**Un array mutable de mensajes con eventos disparados como notificaciones** — más simple, pero el estado y el registro pueden divergir; con event sourcing el registro ES el estado, por lo que la divergencia es estructuralmente imposible.

## Consequences

- Replay, traza y telemetría están garantizados estructuralmente, no añadidos.
- La persistencia permanece una preocupación de plugins; el almacén en memoria se envía en dsh-session.
- El vocabulario de eventos es extensible por fusión (los plugins agregan, por ejemplo, eventos de compactación); [persistencia de sesión](2026-06-14-session-persistence.md) congeló su forma una vez que el registro se volvió durable.
- El costo de derivación crece con la longitud del registro — la compactación (dsh-compaction) es la mitigación prevista, no la mutación del registro.