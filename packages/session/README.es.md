# session/ — plano de datos de sesión duradero

[English](README.md) | Español

La familia duradera en torno al servicio en memoria en vivo de `core/session`: el seam de persistencia con sus backends de almacenamiento y su política de checkpoints, el seam de proyección que sirve valores completos derivados del log, títulos respaldados por el log y telemetría de sesión saliente. Todos son paquetes **producto**. `session-query/` sigue siendo un grupo hermano: la surface de lectura/herramientas se consume con independencia de los internals de la persistencia.

## Persistencia

Persistencia de sesión duradera, política de checkpoints semántica y los backends de almacenamiento distribuidos.

| Paquete | Rol | clave ctx |
|---|---|---|
| [`session-persistence/`](session-persistence/README.es.md) | Define el servicio de persistencia y la coordinación compartida de escritura | `ctx.sessionPersistence` |
| [`session-checkpoint-policy/`](session-checkpoint-policy/README.es.md) | Aplica checkpoints de durabilidad semántica | envuelve `ctx.llm` y `ctx.tools` |
| [`session-persistence-jsonl/`](session-persistence-jsonl/README.es.md) | Persiste sesiones en archivos JSONL | se registra en `ctx.sessionPersistence` |
| [`session-persistence-sqlite/`](session-persistence-sqlite/README.es.md) | Backend SQLite opcional con filas físicas de fragmentos empaquetadas | se registra en `ctx.sessionPersistence` |

La [decisión de session-persistence](../../.agents/notes/implemented/architecture/2026-06-14-session-persistence.md) registra el diseño de la persistencia.

## Proyección

Sirve el estado actual por sesión derivado del log a los carriers de cliente.

| Paquete | Rol | clave ctx |
|---|---|---|
| [`session-projection/`](session-projection/README.es.md) | Define y conduce las unidades de proyección de sesión | `ctx.sessionProjections` |
| [`session-projection-cache/`](session-projection-cache/README.es.md) | Persiste y restaura checkpoints de proyección | `ctx.sessionProjectionCache` |
| [`session-stats/`](session-stats/README.es.md) | Sirve recuentos de conversación de todo el log y tiempos de pared (unidad `sessionStats`) | se registra en `ctx.sessionProjections` |

## Títulos

Deriva títulos de sesión duraderos del log de sesión, con un provider opcional respaldado por modelo.

| Paquete | Rol | clave ctx |
|---|---|---|
| [`session-title/`](session-title/README.es.md) | Es dueño del estado de títulos, del comportamiento de fallback, del registro de providers y de la actualización | `ctx.sessionTitle` |
| [`session-title-llm/`](session-title-llm/README.es.md) | Proporciona generación de títulos compartida respaldada por modelo | — |
| [`session-title-first-prompt-llm/`](session-title-first-prompt-llm/README.es.md) | Pone título a una sesión desde su primer mensaje humano elegible | se registra en `ctx.sessionTitle` |
| [`session-title-all-prompts-llm/`](session-title-all-prompts-llm/README.es.md) | Pone título a una sesión desde todos los mensajes humanos elegibles | se registra en `ctx.sessionTitle` |

Los despliegues pueden registrar un único provider respaldado por modelo; el servicio conserva un fallback determinista cuando no hay ninguno.

## SessionTelemetryBackend

Proyecta la actividad de sesión en telemetría saliente y delega la entrega en un backend de informes configurado. La [decisión de telemetría](../../.agents/notes/implemented/feature/2026-07-23-session-telemetry-otel-revival.md) registra la frontera de informes; la [decisión de modo](../../.agents/notes/implemented/feature/2026-08-05-feedback-gated-session-telemetry.md) registra la entrega inmediata, la limitada por feedback y la deshabilitada.

| Paquete | Rol |
|---|---|
| [`session-telemetry/`](session-telemetry/README.es.md) | Define la captura, la redacción, la proyección y la entrega del backend en vivo o bajo demanda. |
| [`session-telemetry-otel/`](session-telemetry-otel/README.es.md) | Entrega telemetría a través de los logs de OpenTelemetry en modo `FULL`, `FEEDBACK_ONLY` o `DISABLED`. |

Las referencias del subsistema: [persistence.es.md](../../docs/subsystems/persistence.es.md), [session-projection.es.md](../../docs/subsystems/session-projection.es.md), [session-title.es.md](../../docs/subsystems/session-title.es.md) y [session-telemetry.es.md](../../docs/subsystems/session-telemetry.es.md). Solo un provider de títulos puede registrarse a la vez; el spine de demo monta el servicio de fallback y deja ambos providers de modelo fuera de la composición por defecto.
