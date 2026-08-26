# attachment/ — familia de capacidad de adjuntos duraderos
[English](README.md) | Español

El seam de adjuntos binarios duraderos y su implementación local de sistema de archivos. Ambos son paquetes de producto.

| Paquete | Rol | clave ctx |
|---|---|---|
| `attachment/` | Referencias de adjuntos inmutables, límites de imagen y servicio de almacenamiento | `ctx.attachments` |
| `attachment-local/` | Almacenamiento privado con direccionamiento por contenido bajo `DSH_HOME` | (se registra en `ctx.attachments`) |

Los borradores de navegador sin enviar quedan deliberadamente fuera de esta capacidad. Los bytes entran en el almacenamiento duradero solo cuando se envía un prompt de usuario o cuando un adaptador de provider confirma salida de modelo estructurada.
