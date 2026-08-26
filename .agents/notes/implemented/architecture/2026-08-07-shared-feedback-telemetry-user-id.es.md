# Agent Note: Identificador de usuario anónimo compartido entre telemetría, feedback y solicitudes a DeepSeek

Status: implemented

[English](2026-08-07-shared-feedback-telemetry-user-id.md) | Español

## Problema

El backend de OpenTelemetry ya persistía un UUID anónimo en `$DSH_HOME/.anonymous-user-id`. `/feedback` ahora necesita informar tanto el id de sesión receptor como un id de usuario para que un operador pueda correlacionar el acuse con los registros exportados. Duplicar o generar esa identidad de forma independiente haría que el usuario informado perdiera sentido, mientras que importarla desde `session-telemetry-otel` haría que un comando directo dependiera de un backend exportador y crearía un ciclo de dependencias cuando la exportación de feedback la monte la telemetría.

La [decisión anterior de anonymous-user-id](../feature/2026-07-31-telemetry-anonymous-user-id.es.md) mantuvo deliberadamente el helper dentro del backend OTel hasta que existió un segundo consumidor real. Feedback se convirtió en ese segundo consumidor. La [identidad de solicitud directa a DeepSeek](../feature/2026-08-11-deepseek-request-user-id-header.es.md) es el tercero.

## Decisión

`@deepseek-ai/dsh-anonymous-user-id` es dueño de `getOrCreateAnonymousUserId()` y del contrato de almacenamiento `$DSH_HOME/.anonymous-user-id`. `session-telemetry-otel` usa el id devuelto como `user.id` del Resource de OpenTelemetry; el acuse de éxito de `/feedback` informa `Feedback recorded for session {sessionId}` seguido de `Anonymous user: {userId}` en una segunda línea; y las solicitudes directas a DeepSeek lo llevan como `x-deepseek-harness-user-id`. El feedback inválido se rechaza antes de resolver el id, y el adaptador de DeepSeek lo resuelve solo después de que las credenciales tengan éxito, así que ni un comando vacío ni un fallo de credenciales crean `.anonymous-user-id`.

La extracción conserva el UUID aleatorio existente, la resolución del home, la memo de proceso, la concurrencia de creación exclusiva, la sustitución por corrupción y la semántica de escritura best-effort.

## Alternativas consideradas

| Rechazada | Motivo |
|---|---|
| Importar el helper desde `session-telemetry-otel` | Acopla feedback a un backend exportador opcional y forma un ciclo de dependencias inverso en cuanto la telemetría exporta feedback |
| Duplicar el helper de persistencia en feedback | Dos implementaciones de un contrato de archivo pueden divergir y competir con semánticas de validación o fallo distintas |
| Generar un id de usuario de feedback separado | El acuse no podría correlacionarse con el Resource de OTel y no cumpliría el propósito del informe |

## Consecuencias

- Un solo home del harness tiene un id anónimo compartido por los acuses de feedback, las exportaciones de telemetría de sesión y las solicitudes directas a DeepSeek.
- El paquete de feedback depende solo de la capacidad de identidad, no del seam de telemetría ni del SDK de OTel.
- El paquete es una librería compartida justificada con tres consumidores; su acompañante de invariantes vacío explica por qué leer el archivo privado no es una comprobación útil de relación en runtime.
- El Agent Note original de anonymous-user-id sigue siendo autoritativo para la semántica de almacenamiento y privacidad, mientras que este Agent Note solo supera su decisión de propiedad confinada a OTel.
