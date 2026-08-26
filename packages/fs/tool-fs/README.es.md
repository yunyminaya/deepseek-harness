# @deepseek-ai/dsh-tool-fs

[English](README.md) | Español

Las **herramientas de sistema de archivos orientadas al modelo** — `read`, `read_image`, `write`, `edit` — y su **ejecutor**. Esta es la capa Consumer de la pila de sistema de archivos: es la dueña de los nombres de las herramientas, los JSON schemas, la validación de argumentos, las secciones de prompt, la **ventana de lectura** y el formato de los resultados. Lee/escribe/edita a través del contrato de provider `ctx.fs` ([`@deepseek-ai/dsh-fs`](../fs)) **directamente**. La política de frescura/observación la aporta un plugin independiente ([`@deepseek-ai/dsh-fs-observation-policy`](../fs-observation-policy)) a través de la puerta de eventos `fs/*`; la herramienta no está acoplada por método a esa política. Con un provider confinante, el servicio compartido de política de sandbox es obligatorio para la ejecución por sesión y la herramienta expone escalada para las mutaciones del sistema de archivos.

```ts ignore-check
// Default deployment: a ctx.fs provider, the policy plugin, then the tools.
await ctx.plugin(LocalFileSystem, { cwd: process.cwd() }) // @deepseek-ai/dsh-fs-local
await ctx.plugin(FsPolicy)                             // @deepseek-ai/dsh-fs-observation-policy (policy gate)
await ctx.plugin(LocalAttachmentStore, { dshHome })       // optional — enables durable read_image results
await ctx.plugin(ToolFs)                                  // this package — read/write/edit, plus read_image with attachments
```

`@deepseek-ai/dsh-fs-observation-policy` es **opcional**: omítelo y las herramientas se ejecutan contra el provider desnudo (escritura/sobrescritura/edición incondicionales, sin estado observado). Se espera que un despliegue que carga estas herramientas cargue también esa política, de modo que el comportamiento sea leer-antes-de-escribir/editar.

`read_image` solo se registra mientras está montado un servicio duradero `ctx.attachments`. La ejecución exige además que el modelo enrutado exacto declare entrada `image`, resuelta mediante `ctx.llm.resolveModelInfo` a partir de la cabecera de la petición más reciente de la sesión y, después, de las opciones del agent.

## Configuración

Todas las claves son opcionales; los valores por defecto son los límites de lectura que se distribuyen.

| Clave | Valor por defecto | Significado |
|---|---|---|
| `readLimit` | `2000` | Líneas por defecto y máximas que devuelve una llamada `read` (el schema de la herramienta lo anuncia como valor por defecto de `limit`). |
| `readMaxLineLength` | `2000` | Caracteres que se conservan por línea antes de truncar (el sufijo nombra el límite). |
| `readMaxBytes` | `51200` | Límite de bytes para las líneas seleccionadas de una llamada `read`; el desbordamiento cierra la ventana con un pie de «capped». |
| `readStreamMinSize` | `10485760` | Los archivos de este tamaño o mayor (o de tamaño desconocido) se transmiten en streaming en lugar de cargarse enteros en memoria. |

## Herramientas (schemas según la [Agent Note de los schemas de herramientas de sistema de archivos](../../../.agents/notes/implemented/feature/2026-06-17-filesystem-tool-schemas.es.md))

| Herramienta | Argumentos | Comportamiento |
|---|---|---|
| `read` | `file_path`, `offset?`, `limit?` | Contenido UTF-8 numerado por líneas con un pie de paginación. `offset` empieza en 1; `limit` toma por defecto el `readLimit` configurado (2000) y lo usa como tope. |
| `read_image` | `file_path` | Lee un archivo PNG/JPEG/WebP/GIF a través del seam acotado de bytes, lo persiste mediante `ctx.attachments.saveImage` y devuelve un bloque de imagen junto a una pequeña envoltura de metadatos. El harness valida y reduce la escala de las imágenes grandes admitidas antes de la siguiente petición al modelo, de modo que el modelo puede leer la fuente directamente sin crear primero una miniatura. Solo tiene éxito cuando el modelo enrutado exacto declara entrada de imagen. |
| `write` | `file_path`, `content` | Crea un archivo o lo reemplaza por completo. Con el plugin de política: sobrescribir un archivo existente exige un `read` previo en la versión sin cambios; crear un archivo nuevo no. Sin él: incondicional. |
| `edit` | `file_path`, `old_string` no vacío, `new_string`, `replace_all?` | Reemplazo literal; se exige una coincidencia única salvo que `replace_all` sea true. Con el plugin de política: exige un `read` previo (cualquier ventana) y que el archivo no haya cambiado desde entonces. Sin él: incondicional. |

Los nombres de los campos usan snake_case para coincidir con Claude Code y con los schemas de herramientas existentes del harness.

Los éxitos estructurados son `read` → `{ path, offset, lines: [{ number, text }], totalLines }`, `read_image` → `{ path, image: { attachmentId, mediaType, bytes, width, height, name?, originalDimensions?: { width, height } } }`, `write` → `{ path, operation: 'create' | 'update', before: string | null, after }` y `edit` → `{ path, before, after }`. `originalDimensions` solo aparece cuando la normalización redujo la escala del ráster enviado y registra su tamaño de entrada con la orientación aplicada. Los renderizadores nativos conservan la lectura numerada y los acuses de mutación que se muestran abajo. `write`/`edit` derivan metadatos de tarjeta de diff reproducibles, y `read` deriva una ventana de tarjeta de lectura reproducible `{ path, offset, lines, totalLines, lang? }`; los valores estructurados locales de la ejecución no se añaden a `tool/result`, mientras que los renderizadores de imagen emiten los bloques de imagen duraderos que registran los logs de resultados.

## La herramienta es el ejecutor; la política es una puerta de eventos

Las herramientas **no** inyectan un servicio de política ni inspeccionan caché alguna. Cada herramienta resuelve la ruta mediante `ctx.fs.resolve(path, { cwd, signal })` —pasando el cwd de sesión del agent llamante (`exec.agent.session.header.cwd`) para que una ruta relativa se resuelva contra el espacio de trabajo de la sesión, igual que `dsh-tool-bash`, y reenviando la cancelación de la herramienta a través de la resolución (consulta la [Agent Note del cwd por sesión](../../../.agents/notes/implemented/architecture/2026-07-02-fs-per-session-cwd.es.md))— y después:

- **read** — un `ctx.fs.stat` (enrutamiento por tipo + tamaño + versión), después `readText`/`streamText`, luego construye la ventana de líneas y por último emite `fs/observed` con un `ctx.emit` simple. (1 stat.)
- **read_image** — valida el argumento, la extensión, la disponibilidad de adjuntos, los tipos de medios del despliegue y la ruta con capacidad de imagen antes de cualquier E/S; después, un `ctx.fs.stat` (registrando una observación `absent` para un objetivo ausente, como `read`), un `ctx.fs.readBytes` acotado limitado al menor de `imageLimits.maxImageBytes` e `imageLimits.maxMessageImageBytes` (el resultado es un mensaje que lleva una imagen), `attachments.saveImage` (direccionado por contenido, de modo que el bloque de imagen referencia un objeto confirmado de forma duradera cuando se añade `tool/result`), y por último `fs/observed`. (1 stat.)
- **write** — `ctx.waterfall('fs/write-intent', target, exec, () => undefined)` para la guarda opcional, después `ctx.fs.writeText(target, content, intent)` y por último `fs/observed`. (0 stat.)
- **edit** — `ctx.waterfall('fs/edit-intent', target, exec, () => undefined)` para la guarda opcional, después `ctx.fs.editText(target, edit, intent)` y por último `fs/observed`. (0 stat.)

La herramienta pasa `exec` (el contexto de ejecución de la herramienta) como `actor` opaco en cada despacho. Las thunks por defecto devuelven `undefined` (el provider desnudo sin restricciones). Cuando `@deepseek-ai/dsh-fs-observation-policy` está cargado, ocupa la única slot de decisión —devolviendo `createIfAbsent`/`replaceIfVersion`/`{ version }` o lanzando `FS_NOT_OBSERVED`— y registra en `fs/observed`. Los errores del backend (`FsError`) y un `FS_NOT_OBSERVED` lanzado fluyen a través de `ToolRuntime.execute()` y se convierten en resultados de herramienta `isError` con su `{ name, code }` adjunto.

Cuando `ctx.fs.sandboxMode` informa confinamiento, write/edit anuncian `sandbox_permissions` y `justification` y resuelven los reintentos aprobados mediante `ctx.approval`. El propietario de la política aporta la política permanente neutral respecto a capacidades; los resultados de las herramientas conservan la orientación de denegación y reintento específica de cada operación.

## `fs/observed` es de disparar y olvidar

`fs/observed` se dispara DESPUÉS de que read/read_image/write/edit ya hayan tenido éxito, mediante un `ctx.emit` simple. Un listener es contractualmente un registrador síncrono de solo efectos secundarios (el de `@deepseek-ai/dsh-fs-observation-policy` es un `WeakMap.set`); la herramienta no protege la emisión, así que un listener que lance una excepción aparecería como resultado `isError` de la herramienta — la observación asíncrona o fallible no pertenece a este evento.

`read` se acoge a la planificación concurrente porque su única mutación es el registrador síncrono de versiones. Las carreras del registrador fallan en modo cerrado cuando un `write` o `edit` posterior vuelve a comprobar la versión bajo su bloqueo de objetivo; ambas herramientas de mutación siguen siendo exclusivas. Consulta la [Agent Note de ejecución paralela de llamadas de herramienta](../../../.agents/notes/implemented/feature/2026-07-10-parallel-tool-call-execution.es.md).

La raíz del paquete exporta solo el contrato de plugin de Cordis (`name`, `inject`, `Config` y `apply`). El renderizado de lectura (ventana de líneas + formato de salida) vive en `src/read-render.ts` (libre de Cordis, probado unitariamente de forma independiente); `src/read.ts`/`read-image.ts`/`write.ts`/`edit.ts` son los ejecutores de las herramientas y `src/index.ts` los compone.

## Experiencia de modelo

### Prompt del sistema

#### Lo que ve el modelo

Cada petición en el ámbito de registro de este plugin recibe la orientación de read, write y edit registrada de forma independiente que se muestra abajo. Las restricciones de herramienta con ámbito pueden ocultar los schemas sin eliminar estas secciones.

##### Orientación de read

```markdown
Use the read tool — not shell commands like cat — to inspect text files. Results include line numbers. Use offset and limit to continue reading large files.
```

##### Orientación de write

```markdown
Use the write tool to create files or completely replace file contents. Existing files are overwritten, so read an existing file first (the default fs-observation-policy requires it) and prefer edit for targeted changes.
```

##### Orientación de edit

```markdown
Use the edit tool for targeted changes to existing UTF-8 text files. It replaces literal old_string with new_string; by default old_string must appear exactly once. If old_string appears multiple times, provide a more specific old_string or set replace_all to true. Read the file first (the default fs-observation-policy requires it), unless you just created or edited it in this session.
```

#### Efecto en tokens

Coste fijo de orientación por petición mientras el plugin está activo, incluso cuando una restricción oculta una o más herramientas.

#### Efecto en la KV cache

Estable en el prefijo mientras el ámbito del plugin y el texto de orientación no cambian. Las restricciones de herramienta no eliminan esta sección, pero la activación o el desmontaje del plugin pueden invalidar la reutilización de ella.

### Schemas de herramienta

#### Lo que ve el modelo

El modelo ve los [schemas generados de `read`, `read_image`, `write` y `edit`](../../../docs/tool-catalog.es.md#deepseek-aidsh-tool-fs), con argumentos snake_case. La herramienta de imagen solo aparece mientras hay un almacén de adjuntos duradero montado; su schema es independiente de la ruta, y la puerta estricta la rechaza en la ejecución. Las restricciones de herramienta con ámbito pueden eliminar cualquier definición para un agent.

#### Efecto en tokens

Coste fijo de schema en cada petición de esa vista de herramienta.

#### Efecto en la KV cache

Estable en el prefijo mientras las definiciones visibles de herramientas y su orden no cambian. El ciclo de vida del registro o las restricciones con ámbito pueden invalidar la reutilización desde el primer token de schema cambiado.

### Resultado de read

#### Lo que ve el modelo

Un read con éxito es exactamente `<path><displayPath></path>`, salto de línea, `<type>file</type>`, salto de línea, `<content>`, las líneas numeradas como `<lineNumber>: <text>`, una línea en blanco, un pie de página y `</content>`. El pie de página es exactamente `(Output capped. Showing lines <start>-<end>. Use offset=<next> to continue.)`, `(Showing lines <start>-<end> of <total>. Use offset=<next> to continue.)` o `(End of file - total <total> lines)`. Una línea larga termina exactamente con `... (line truncated to <max> chars)`. Un read ausente sigue devolviendo `FS_NOT_FOUND`, pero registra la ausencia confirmada para la sesión llamante; después de releer un archivo eliminado externamente, un `write` reintentado puede recrearlo con seguridad a través de la guarda de no-reemplazo del provider.

#### Efecto en tokens

La salida de read está limitada por `readLimit`, `readMaxLineLength` y `readMaxBytes`; la llamada y el resultado retenidos se reenvían hasta la compactación.

#### Efecto en la KV cache

Solo añadidura; el contenido recién visible sigue el prefijo de petición reutilizable y no invalida las entradas KV-cache existentes.

### Resultado de lectura de imagen

#### Lo que ve el modelo

Un `read_image` con éxito devuelve `<path><displayPath></path>`, `<type>image</type>` y una envoltura `<content>` que nombra el tipo de medio, las dimensiones normalizadas y el tamaño en bytes, seguida de la imagen en sí como bloque de imagen nativo. El resultado se registra con su referencia duradera antes de la siguiente petición al modelo.

#### Efecto en tokens

La imagen se factura en cada petición posterior hasta la compactación. Cada llamada está acotada de forma independiente por `maxImageBytes`/`maxImagePixels`/`maxImageDimension` del almacén de adjuntos; las llamadas con éxito repetidas acumulan historial, y el direccionamiento por contenido deduplica solo los bytes almacenados, no el coste de tokens por petición.

#### Efecto en la KV cache

Solo añadidura; el contenido recién visible sigue el prefijo de petición reutilizable y no invalida las entradas KV-cache existentes.

### Resultados de write y edit

#### Lo que ve el modelo

Write devuelve la envoltura exacta de cinco líneas `<path><displayPath></path>`, `<type>file</type>`, `<content>`, `Created file` o `Updated file` y, por último, `</content>`. Edit devuelve exactamente `The file <displayPath> has been updated successfully.` o, para `replace_all`, `The file <displayPath> has been updated. All occurrences were successfully replaced.` El texto completo de write o del reemplazo permanece en los argumentos de la llamada de herramienta del asistente.

#### Efecto en tokens

El texto de éxito es pequeño, pero los argumentos de mutación grandes y cualquier resultado se reenvían hasta la compactación.

#### Efecto en la KV cache

Solo añadidura; el contenido recién visible sigue el prefijo de petición reutilizable y no invalida las entradas KV-cache existentes.

### Errores de herramienta

#### Lo que ve el modelo

Los fallos se normalizan como `Error: <message>`. Los mensajes estables de validación y lectura de este paquete son `file_path must be a non-empty string`, `limit must be less than or equal to <max>`, `old_string must be a non-empty string`, `old_string and new_string must differ`, `cannot read "<path>": not found`, `cannot read "<path>": not a regular file`, `offset <offset> is out of range for "<path>" (<total> lines)`, `cannot read "<path>": read_image only accepts PNG/JPEG/WebP/GIF paths`, `cannot read "<path>" as an image: model "<model>" does not declare image input; switch to an image-capable model to read images` y la reparación de desajuste `cannot read "<path>": the <ext> extension declares <type>, but the bytes use a different image format; rename the file to match its actual format if it is PNG/JPEG/WebP/GIF, or convert it to one of those formats`. Una conversión de 16 bits fallida informa `cannot read "<path>": the 16-bit PNG could not be converted to the normalized 8-bit sRGB form; convert it to an 8-bit PNG/JPEG/WebP and retry`. Las plantillas del provider y de la política se citan en sus README de paquete. Los fallos de mutación con guarda llevan además su instrucción de recuperación en el mensaje, añadida por el envoltorio de errores orientado al modelo de este paquete: `FS_STALE_VERSION` recibe `— re-read the file, then retry`, y `FS_NOT_OBSERVED` recibe `— read the file, then retry`; el código estructurado se conserva. Cuando esa relectura confirma la ausencia, edit informa `FS_NOT_FOUND` en lugar de repetir un remedio obsoleto, mientras que write usa la creación con guarda.

#### Efecto en tokens

Solo una llamada fallida añade esos tokens retenidos.

#### Efecto en la KV cache

Solo añadidura; el contenido recién visible sigue el prefijo de petición reutilizable y no invalida las entradas KV-cache existentes.

## Limitaciones conocidas y trabajo pendiente

- **No se distribuye un listado de directorios orientado al modelo** — `ctx.fs.listDir` sirve al código del provider, como el descubrimiento de skills, mientras que el paquete hermano [`dsh-tool-fs-search`](../tool-fs-search/) aporta `glob` y `grep` basados en ripgrep en lugar de extender el seam del sistema de archivos.
- **`read` solo maneja archivos de texto UTF-8** — las imágenes usan la herramienta independiente `read_image` enrutada por extensión; PDF, audio y vídeo siguen pendientes. Un objetivo de directorio es `FS_NOT_REGULAR_FILE`.
- **Tipo de medio declarado por la extensión** — la extensión selecciona el tipo declarado y la validación de magic bytes del almacén de adjuntos sigue siendo autoritativa; una imagen con formato correcto bajo una extensión equivocada se rechaza con el remedio de renombrar en lugar de deducirse por su contenido.
- **Sin vista previa de imagen en línea en la tarjeta de resultado de herramienta** — las superficies de UI renderizan el resultado de imagen de forma genérica (la referencia duradera, no los píxeles); el renderizado en línea queda diferido a los paquetes de UI.
- **Sin herramienta de región de adjunto** — un agent puede recortar una imagen mediante otras herramientas disponibles cuando tiene una ruta de sistema de archivos. Una imagen pegada o arrastrada sin ruta no se puede releer a mayor resolución.
- **Sin superficie de timeout** — `read`/`write`/`edit` no aceptan argumento de timeout ni declaran presupuesto `timeout-policy`; la cancelación viaja solo con `exec.signal` ([fundamento del provider](../README.es.md#no-timeouts-on-file-io)).
