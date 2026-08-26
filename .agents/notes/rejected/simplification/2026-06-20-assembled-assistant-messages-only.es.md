# Agent Note: Persistir mensajes de assistant ensamblados, no fragmentos del stream

Status: rejected — el replay de fragmentos de alta fidelidad, los streams fallidos parciales y el replay de instantáneas dependen hoy de los eventos `assistant/chunk` persistidos. Descartar los chunks solo es viable con un reemplazo de replay/artefactos sin pérdida de información.

[English](2026-06-20-assembled-assistant-messages-only.md) | Español

## Problema

El log de sesión canónico persiste hoy cada `assistant/chunk` exactamente como lo emite el modelo en el stream. El [Agent Note de persistencia de sesión](../../implemented/architecture/2026-06-14-session-persistence.es.md) lo eligió por la fidelidad de replay a nivel de token y por la `seq` contigua, pero el coste ha crecido: los fixtures JSONL están dominados por registros de delta diminutos, los escenarios de instantánea reproducen el modelo agrupando eventos de chunk, la carga de ACP reconstruye la salida previa del assistant desde los chunks, y cualquier lector de logs futuro debe distinguir el historial de mensajes durable del rastro a nivel de token.

Para los pasos correctos que ensamblan contenido completo, el loop ya añade un `assistant/message`. Ese es el evento que `deriveMessages()` usa para la siguiente petición al modelo. En otras palabras, el estado normal de conversación reanudable ya está presente sin los chunks; los chunks son un artefacto de render en vivo y de pruebas deterministas, no historial de conversación requerido. Los streams fallidos o abortados son diferentes: la salida parcial del assistant puede existir solo como chunks, y los pasos max-token vacíos pueden no producir ningún `assistant/message`.

## Propuesta

Dejar de almacenar `assistant/chunk` en el log de sesión canónico. El log durable conserva `assistant/message`, `tool/call`, `tool/result`, `usage` si se retiene, y los límites de turno. Las UIs en vivo pueden seguir recibiendo deltas de tokens a través de un evento de stream deliberadamente transitorio. El replay de instantáneas debe mover su script de modelo a un fixture sidecar explícito o derivarlo de un artefacto de adaptador grabado, en lugar de tratar la sesión de usuario canónica como una cinta de tokens. Los escenarios que necesitan salida de streams fallidos parciales deben registrar esa salida en el fixture de replay.

ACP `session/load` puede reproducir los mensajes previos del assistant como bloques de contenido completos en lugar de simular el stream de tokens original. Un transcript cargado no necesita reproducir cada delta histórico; debe mostrar el mismo contenido de assistant completado y reanudar con un historial de provider válido.

## Criterios de aceptación

- `SessionEventMap` elimina `assistant/chunk`, o lo marca como no persistido si se necesita un evento en vivo transitorio.
- [Docs de persistencia de sesión](../../../../packages/session/session-persistence/README.es.md) ya no exigen almacenar cada chunk del stream verbatim.
- `llm-replay` y las instantáneas de ACP usan un formato de fixture de replay explícito o sidecar para los chunks del modelo.
- `session/load` renderiza los mensajes de assistant completados desde `assistant/message`.
- Los logs almacenados se vuelven mucho más pequeños y siguen siendo `seq`-contiguos sin huecos de chunk.
- Se refrescan la versión del formato de sesión y los fixtures grabados; los logs almacenados no actuales se rechazan según la política de formato pre-release.

## Qué renunciamos

La sesión de usuario canónica ya no reconstruye el stream de tokens exacto de un turno antiguo. También pierde la salida parcial del assistant de streams fallidos o abortados a menos que otro evento o fixture la registre. Es demasiada pérdida de información para los contratos actuales de reanudación, carga e instantáneas. Las pruebas que necesitan streams deterministas exactos deben ser dueñas de ese fixture directamente solo si el log de sesión de producción conserva suficiente fidelidad para la recuperación visible por el usuario.

## Relacionado

Esto supera la elección de persistencia de chunks en [persistencia de sesión](../../implemented/architecture/2026-06-14-session-persistence.es.md) y afecta a [pruebas de instantánea ACP](../../implemented/testing/2026-06-19-acp-snapshot-tests.es.md), cuyo plugin de replay actual deriva su script de los eventos `assistant/chunk`.

<!-- agent-note-format: alternatives-not-recorded (pre-format Agent Note) -->
