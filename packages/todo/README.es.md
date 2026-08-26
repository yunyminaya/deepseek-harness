# todo/ — familia de capacidades de todo / planificación

[English](README.md) | Español

La capacidad de todo orientada al modelo. Es un único paquete de **producto** porque una sesión de agent es dueña de la lista; no hay contrato de provider reemplazable.

| Paquete | Rol | Clave de ctx |
|---|---|---|
| [`tool-todo/`](tool-todo/README.es.md) | Almacena y expone la lista de todos de la sesión. | (se registra en `ctx.tools`) |

El README hijo es dueño del contrato de la herramienta, la persistencia y el renderizado.

El payload del evento se documenta en [docs/subsystems/session.md](../../docs/subsystems/session.es.md).
