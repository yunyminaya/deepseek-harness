# Agent Note: Los errores de mutación vigilada añaden la instrucción de recuperación en el límite del modelo

Status: implemented

[English](2026-08-03-fs-tool-error-remedy.md) | [中文](2026-08-03-fs-tool-error-remedy.zh.md) | Español

## Problema

Los fallos vigilados de `write` y `edit` llegan al modelo con mensajes que declaran la condición pero no la única recuperación correcta: `FS_STALE_VERSION` («el archivo cambió desde que se leyó») y `FS_NOT_OBSERVED` («editar requiere leer … primero»). El modelo debe adivinar que la recuperación es una relectura (o una primera lectura) seguida de un reintento, y las capas de reintento/permiso/UI que enrutan sobre el código estructurado ven el mismo texto de mensaje. Los mensajes propiedad del provider son parte del vocabulario orientado a máquina del seam de almacenamiento ([seam de capacidad de filesystem](../architecture/2026-06-17-filesystem-capability-seam.es.md)), así que el remedio no puede vivir allí sin filtrar redacción orientada al modelo a cada consumidor de `FsError`.

## Decisión

`dsh-tool-fs` posee un wrapper de errores orientado al modelo, `remediateFsError` en `src/error.ts`, aplicado en `write.ts` y `edit.ts` después del mapeo de denegación de sandbox. Añade la instrucción de recuperación a los dos códigos de mutación vigilada y pasa todo lo demás intacto:

- `FS_STALE_VERSION` (incluido un objetivo de edición ausente, que comparte el código de obsolescencia) gana «— vuelve a leer el archivo, luego reintenta».
- `FS_NOT_OBSERVED` gana «— lee el archivo y reintenta».

El código estructurado de `FsError` se conserva para que las capas de reintento/permiso/UI sigan enrutando sobre él, y el error original encadena como `cause`. Los mensajes del provider siguen siendo orientados a máquina y sin cambios.

En `edit.ts` el waterfall (cascada de eventos) `fs/edit-intent` ahora está dentro del mismo `try` que la mutación del provider, así que el rechazo `FS_NOT_OBSERVED` de la política lanzado desde el slot de intento también recibe el remedio — ambas rutas de rechazo llegan al modelo con la misma redacción de recuperación.

## Alternativas consideradas

- **Añadir el remedio a los mensajes del provider en `dsh-fs` / `dsh-fs-local`.** Rechazada porque esos mensajes son vocabulario de seam orientado a máquina consumido por las capas de reintento, permiso, UI y orientadas al modelo; la redacción orientada al modelo pertenece al límite del modelo, donde `dsh-tool-fs` ya posee el formato de resultados ([seam de capacidad de filesystem](../architecture/2026-06-17-filesystem-capability-seam.es.md)).
- **Añadir la recuperación a la guía de prompt en su lugar.** Rechazada porque el fallo llega a mitad de tarea; una instrucción estática no alcanza de forma fiable la decisión de reintento, mientras que el mensaje de error está presente exactamente cuando el modelo debe actuar.
- **Señalar el remedio con un código nuevo de `FsError`.** Rechazada porque los dos fallos son las mismas condiciones que las capas de reintento ya manejan; dividir el código bifurcaría el enrutado sobre semánticas idénticas.

## Consecuencias

El texto visible para el modelo de los dos códigos cambia; la instantánea sin clave `fs-policy-reject` se vuelve a grabar, y los README de `dsh-tool-fs` y `dsh-fs-observation-policy` fijan el texto añadido exacto. Las pruebas unitarias cubren el wrapper directamente (texto del remedio, preservación del código, encadenamiento de causa, passthrough de otros códigos y de valores que no son `FsError`) y las rutas de herramienta ensambladas afirman que el remedio llega al modelo para ambos códigos.

El [seguimiento de observación de ausencia del filesystem](../bug-fix/2026-08-09-filesystem-absence-observation.es.md) hace accionable el remedio de obsolescencia para la eliminación externa. La relectura fallida sigue devolviendo `FS_NOT_FOUND`, pero registra la ausencia confirmada: `edit` devuelve entonces `FS_NOT_FOUND` sin otro remedio de obsolescencia, mientras que `write` reintenta como un `createIfAbsent` atómico y preserva a cualquier creador concurrente.
