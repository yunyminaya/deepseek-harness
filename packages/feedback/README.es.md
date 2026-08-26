# feedback/ — comentarios humanos registrados

[English](README.md) | Español

La familia feedback expone dos contratos deliberadamente separados: una observación inmutable en el registro canónico de la sesión, y comentarios editables adjuntos a un mensaje de asistente concreto en un sidecar local. Ninguna de las dos formas entra en la conversación del modelo.

| Paquete | Rol | Clave ctx |
|---|---|---|
| `command-feedback/` | Evento `feedback/record` independiente del desencadenante más el productor `/feedback` orientado a humanos | — |
| `message-feedback/` | Sidecar de valoración/nota por mensaje ligado al ciclo de vida más el contrato Remote `messageFeedback.list/put/delete` del Host | `messageFeedback` |

Una observación de feedback de comando es solo de registro: nunca entra en el contexto del modelo ni en el historial derivado. Cuando está montado, [`dsh-session-telemetry-otel`](../session/session-telemetry-otel) observa `feedback/record` para liberar un prefijo de telemetría pendiente o advertir de que la telemetría deshabilitada deja el feedback en local; la captura en sí permanece independiente de esa política.

El feedback de mensaje no es un evento de sesión ni una proyección. Permanece en el sidecar del dominio de almacenamiento y no provoca ninguna transferencia de telemetría. El contrato Remote del Host se distribuye con el servicio; el montaje del agregado Remote del cliente y el Consumer de interfaz de usuario se poseen por separado y quedan diferidos.
