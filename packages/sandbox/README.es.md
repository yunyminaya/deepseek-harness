# sandbox/ — familia de capacidades de sandbox de procesos

[English](README.md) | Español

Esta familia aplica política de confinamiento por sesión a la ejecución de procesos. Cubre subprocesos del mismo mundo; los entornos aislados reemplazan implementaciones completas de capacidad en lugar de registrarse aquí.

| Paquete | Rol | Clave ctx |
|---|---|---|
| [`sandbox/`](sandbox/README.es.md) | Define el servicio de sandbox de procesos y el vocabulario compartido de escalada | `ctx.sandbox` |
| [`sandbox-local/`](sandbox-local/README.es.md) | Aporta los backends de confinamiento de plataforma local | se registra en `ctx.sandbox` |
| [`sandbox-policy/`](sandbox-policy/README.es.md) | Resuelve la política de sandbox durable por sesión | `ctx.sandboxPolicy` |

Consulta la [decisión de sandbox](../../.agents/notes/implemented/feature/2026-07-06-sandbox.es.md) para el límite de la capacidad y la [decisión de integración del sistema de archivos](../../.agents/notes/implemented/feature/2026-07-14-cross-family-fs-sandbox.es.md) para el uso de política entre familias.

La referencia del subsistema — modos y enforcement, política por llamada, dialectos de argv envuelto, errores fail-closed — está en [docs/subsystems/sandbox.md](../../docs/subsystems/sandbox.es.md); el límite y la fase entre familias viven en las Agent Notes de [sandbox](../../.agents/notes/implemented/feature/2026-07-06-sandbox.es.md) y [sandbox de fs entre familias](../../.agents/notes/implemented/feature/2026-07-14-cross-family-fs-sandbox.es.md).
