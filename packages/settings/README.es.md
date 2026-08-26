# settings/ — familia de capacidad de ajustes de usuario

[English](README.md) | Español

Esta familia resuelve la configuración editable por el usuario a través de namespaces registrados y providers de almacenamiento intercambiables.

| Paquete | Rol | Clave ctx |
|---|---|---|
| [`settings/`](settings/README.es.md) | Define el registro de namespaces, la resolución por capas y las confirmaciones | `ctx.settings` |
| [`settings-file/`](settings-file/README.es.md) | Almacena los ajustes en un archivo local y observa las ediciones externas | se registra en `ctx.settings` |

La referencia del subsystem — namespaces, alcances de propietario, orden de resolución, confirmaciones en caliente — es [docs/subsystems/settings.es.md](../../docs/subsystems/settings.es.md).
