# @deepseek-ai/dsh-spill-local

[English](README.md) | Español

La implementación de **sistema de archivos local** del [seam de almacenamiento `@deepseek-ai/dsh-spill`](../spill). Se registra como `ctx.spillStore` y persiste el texto sobredimensionado de una herramienta en un archivo privado con alcance de sesión; su localizador es la ruta del archivo y su pista de recuperación le dice al modelo que use `read` o `grep` sobre esa ruta.

## Disposición del almacenamiento

Los archivos aterrizan en `<root>/session-<hash>/<random>-<safeName>`:

- **`root`** — el `root` de la configuración (resuelto a absoluto), o un directorio privado (0700) por proceso, creado de forma perezosa, bajo el directorio temporal del SO cuando se omite. Una raíz predecible y legible por todo el mundo permitiría a otros usuarios locales leer el spill de salida de herramientas o plantar symlinks.
- **`session-<hash>`** — un prefijo corto de `sha256(sessionId)`, de modo que los archivos de spill de una sesión se agrupen y una limpieza futura pueda eliminarlos por sesión.
- **`<random>-<safeName>`** — un prefijo hexadecimal impredecible (frustra la plantación de symlinks en una raíz compartida) más el `suggestedName` del llamador saneado a un único segmento de ruta seguro (a prueba de traversal; refleja el `encodeSegment` del backend de persistencia JSONL). La escritura es exclusiva y solo del propietario (`open(path, 'wx', 0o600)`): falla ante cualquier ruta preexistente, sea symlink o no, de modo que un destino plantado no pueda redirigirla.

## Configuración

| Clave | Valor por defecto | Significado |
|---|---|---|
| `root` | directorio temporal privado 0700 | Directorio raíz de los archivos de spill. Configúralo para mantenerlos bajo una ubicación conocida. |

`saveText` rechaza ante un fallo real de almacenamiento (permisos, ENOSPC); la política de spill trata el rechazo como best-effort y conserva el resultado inline. Consulta el README del seam para el vocabulario y la [Agent Note de spill de salida de herramientas](../../../.agents/notes/implemented/architecture/2026-07-08-tool-output-spill-files.es.md) para el diseño.

## Experiencia del modelo

De forma indirecta, a través de los consumidores de spill que renderizan la ruta local y la guía de recuperación `read`/`grep`.

#### Efecto de KV Cache

Sin invalidación directa; el consumidor nombrado es dueño de cualquier cambio de prefijo de solicitud.

## Limitaciones conocidas y trabajo diferido

- **Los archivos de spill locales persisten hasta que una limpieza externa los elimine** — el backend no tiene borrado ligado al ciclo de vida de la sesión ni política de retención por antigüedad, porque las sesiones persistidas, reanudadas y bifurcadas pueden seguir referenciando una ruta.
- **Los localizadores requieren un consumidor de sistema de archivos en el mismo host** — un despliegue remoto o virtual necesita otro backend de `SpillStore` cuyo localizador y pista de recuperación tengan sentido allí.
