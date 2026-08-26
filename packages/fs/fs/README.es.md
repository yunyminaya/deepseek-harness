# @deepseek-ai/dsh-fs

[English](README.md) | Español

El **`FileSystem`** (`ctx.fs`) define las primitivas de almacenamiento en un único mundo de ejecución — resolver rutas, exponer rutas de proceso canónicas y URIs de archivo, comprobar contención, leer texto completo o transmitido, leer bytes en bruto acotados, inspeccionar/listar metadatos, escribir atómicamente y aplicar una edición literal — sin decir CÓMO. Ambas mutaciones aceptan su guardia de versión de forma **opcional**, de modo que `ctx.fs` por sí solo es un seam de almacenamiento completo y sin restricciones. Este paquete también es propietario del vocabulario de eventos de política `fs/*` que la herramienta despacha y que el plugin de política escucha.

Este paquete es propietario de la capa de Service Definition y contrato de provider de la pila de sistema de archivos de cuatro capas, dividida para que cada preocupación pueda evolucionar (e intercambiarse) de forma independiente (ver [el Agent Note del seam de capacidad](../../../.agents/notes/implemented/architecture/2026-06-13-capability-seams.es.md), [el Agent Note del seam de capacidad del sistema de archivos](../../../.agents/notes/implemented/architecture/2026-06-17-filesystem-capability-seam.es.md), [el Agent Note de dividir el seam del sistema de archivos](../../../.agents/notes/implemented/simplification/2026-06-26-fsspec-style-fs-seam.es.md) y [el Agent Note de la compuerta de eventos de contexto de archivo](../../../.agents/notes/implemented/architecture/2026-06-26-file-context-as-event-gate.es.md)):

| Capa | Paquete | Rol |
|---|---|---|
| herramienta / ejecutor | `@deepseek-ai/dsh-tool-fs` | schemas `read`/`write`/`edit` orientados al modelo + ventana de lectura + renderizado de texto; lecturas/escrituras/ediciones mediante `ctx.fs`, despacha los eventos `fs/*` |
| política | `@deepseek-ai/dsh-fs-observation-policy` | estado observado + lectura-antes-de-editar + escritura/edición protegida por versión, contribuida a través de la compuerta de eventos `fs/*` (sin servicio) |
| contrato de provider | `@deepseek-ai/dsh-fs` (este) | `ctx.fs`: rutas del mundo de ejecución, E/S de texto y primitivas de mutación atómica (guardia de versión opcional); es propietario del vocabulario de eventos `fs/*` |
| provider | `@deepseek-ai/dsh-fs-local` | la implementación del sistema de archivos host |

`fs-sandbox` y `fs-e2b` implementan esta interfaz sin tocar las capas de política/herramientas.

## API de servicio (`ctx.fs`)

Un backend subclasifica `FileSystem` e implementa doce primitivas.

| Miembro | Semántica |
|---|---|
| `resolve(path, opts?)` | Resuelve una ruta en un `FsTarget` estable (`targetKey` opaco, `displayPath`). `opts.cwd` es la base contra la que se resuelve una `path` relativa (un llamador aporta su workspace de sesión; las rutas absolutas lo ignoran; omitido ⇒ el valor por defecto del backend), mientras que `opts.signal` aborta una ida y vuelta del backend. Asíncrono — un backend remoto puede necesitar E/S. El mismo archivo a través de rutas distintas debe producir el mismo `targetKey`. |
| `processPath(target)` | Devuelve la ruta absoluta canónica que un subproceso en el mundo de ejecución de este provider puede abrir. Es deliberadamente distinta del `targetKey` opaco. |
| `fileUrl(target)` | Devuelve la URI `file:` canónica en la sintaxis de plataforma del mundo de ejecución. El backend, no el proceso host, es propietario de la codificación. |
| `contains(parent, child)` | Comprueba identidad/contención descendente canónica sin exponer ni analizar claves de destino. Ambos objetivos provienen de este provider. |
| `stat(target, signal?)` | Devuelve los metadatos `FsInfo` (`version`, `type`, `size` opcional), o `undefined` cuando el objetivo está ausente. Nunca contenido. |
| `lstat(path, opts?, signal?)` | Devuelve los metadatos `FsPathInfo` sin seguir el último componente de la ruta cuando es un enlace simbólico. Tiene forma de ruta para que los consumidores puedan rechazar enlaces simbólicos propiedad del repositorio antes de que `resolve` los siga hasta un objetivo. |
| `readText(target, signal?)` | Lee el archivo de texto normal completo como una sola cadena decodificada. Es propietario de las comprobaciones de archivo regular, la decodificación UTF-8 y el rechazo binario/NUL (`FS_NOT_TEXT`). |
| `streamText(target, signal?)` | Transmite el mismo texto como fragmentos decodificados para archivos grandes (la decodificación UTF-8 entre fragmentos se queda aquí); los consumidores que necesitan un techo de bytes lo aplican mientras consumen el flujo. |
| `readBytes(target, signal, maxBytes)` | Lee un archivo regular completo como bytes en bruto sin decodificación ni rechazo binario. `maxBytes` es obligatorio y acota el contenido completo en este seam: un desbordamiento conocido o descubierto falla con `FS_TOO_LARGE` en lugar de truncar o almacenar en búfer sin límite. |
| `listDir(target, signal?)` | Lista los hijos directos del directorio en orden de nombre estable. Devuelve nombres de entrada, tipos de entrada, objetivos hijos resueltos y metadatos baratos (`version`/`size` de archivo cuando están disponibles); nunca lee contenidos de archivos. Los objetivos ausentes lanzan `FS_NOT_FOUND`, los no-directorios lanzan `FS_NOT_DIRECTORY`, los fallos de permiso lanzan `FS_PERMISSION_DENIED` y otros fallos de E/S del backend lanzan `FS_IO_ERROR`. Los hijos rotos/desaparecidos pueden devolverse como `other` sin metadatos; los fallos de permiso/E/S de un hijo hacen fallar todo el listado con los mismos códigos estructurados. |
| `writeText(target, content, expected?, signal?)` | Creación/reemplazo atómico. `expected` es OPCIONAL: omitir ⇒ creación-o-sobrescritura incondicional; aportar un `FsWriteIntent` (`createIfAbsent`/`replaceIfVersion`) para proteger. `createIfAbsent` debe realizar una publicación sin reemplazo para que un creador que compita con la sonda inicial se preserve. |
| `editText(target, edit, expected?, signal?)` | Edición literal. `expected` es OPCIONAL: omitir ⇒ edición incondicional del contenido actual; aportar `{ version }` para proteger (verificado ANTES de la coincidencia). Un objetivo ausente informa `FS_STALE_VERSION` en cualquier caso. Aplica y escribe atómicamente — una única sección crítica de mutación. |

La mutación se ejecuta dentro del bloqueo por objetivo del backend en cualquier caso, de modo que una escritura/edición incondicional sigue siendo atómica — «incondicional» elimina la precondición de *versión*, no la atomicidad.

## Los eventos de política `fs/*`

Este paquete declara tres eventos (ver la región generada de [filesystem.md](../../../docs/subsystems/filesystem.es.md#cordis-surface)) para que el emisor (`@deepseek-ai/dsh-tool-fs`) y el listener de política (`@deepseek-ai/dsh-fs-observation-policy`) compartan un vocabulario sin que el emisor dependa del plugin de política. `fs/write-intent` y `fs/edit-intent` son waterfalls de decisión de un solo slot (el listener decide por completo, sin llamar nunca a `next()`); `fs/observed` es un evento de registro de dispara-y-olvida que transporta una unión discriminada `FsObservation`: presente con una versión o ausencia confirmada. Transportan solo vocabulario de `dsh-fs` además de un actor `object` opaco — sin conceptos orientados al modelo y sin estructura de propietario agent (agente)/sesión.

## Un contrato de provider, no la capa de política

`ctx.fs` está deliberadamente cerca de las primitivas de almacenamiento estilo fsspec — medio nivel por encima de `cat`/`open` a nivel de bytes, porque decodifica texto y rechaza binarios para que la capa de política nunca toque bytes en bruto. Es propietario de la decodificación UTF-8, el rechazo binario, las escrituras atómicas y la sección crítica de la edición literal. **No** es propietario de ventanas de líneas, líneas numeradas, pies de página renderizados ni estado observado. El estado observado, la lectura-antes-de-editar y la escritura/edición protegida por versión son política que un plugin (`@deepseek-ai/dsh-fs-observation-policy`) AÑADE al aportar la guardia opcional — no comportamiento del provider — de modo que un backend con sandbox/remoto no hereda ninguna política de observación orientada al modelo.

`editText` permanece en este seam (no compuesto en la capa de política a partir de una lectura más una escritura) porque la guardia de versión + la coincidencia literal + la reescritura atómica deben permanecer dentro de una única sección crítica para una atribución de errores correcta y una concurrencia de uno-gana/uno-obsoleto, y un backend remoto puede implementarlo como un comparar-y-editar nativo.

## Vocabulario

`FsTargetKey` / `FsVersion` son ids opacos marcados ([el Agent Note de ids marcados](../../../.agents/notes/implemented/architecture/2026-06-20-branded-ids.es.md)) — los consumidores no deben analizar `targetKey` ni interpretar `version`; solo `displayPath` es para salida de modelo/UI. `FsObservation` distingue `{ kind: 'present', version }` de `{ kind: 'absent' }`, de modo que una política puede separar un objetivo no visto de una ausencia confirmada sin realizar E/S. `FsWriteIntent` es la intención de escritura PROTEGIDA explícita (`createIfAbsent` crea un objetivo ausente y rechaza uno existente con `FS_NOT_OBSERVED`; `replaceIfVersion` reemplaza solo en la versión observada, si no `FS_STALE_VERSION`); omitirlo de `writeText` es el tercer estado, incondicional. `FsPathInfo` es la forma de metadatos sin-seguir que puede informar `symlink`, a diferencia de `FsInfo` a nivel de objetivo. Los fallos lanzan `FsError` (extiende `HarnessError`, [el Agent Note de la taxonomía de errores estructurados](../../../.agents/notes/implemented/architecture/2026-06-11-structured-error-taxonomy.es.md)) que transporta un `FsErrorCode` estable (`FS_NOT_FOUND`, `FS_NOT_DIRECTORY`, `FS_NOT_TEXT`, `FS_NOT_REGULAR_FILE`, `FS_TOO_LARGE`, `FS_PERMISSION_DENIED`, `FS_IO_ERROR`, `FS_STALE_VERSION`, `FS_NOT_OBSERVED`, `FS_AMBIGUOUS_EDIT`, `FS_EDIT_NOT_FOUND`, `FS_ABORTED`); el registro de herramientas expone `{ name, code }` en los resultados `isError`. Ver `src/types.ts` para los contratos completos.

## Experiencia del modelo

Indirectamente, a través de `dsh-tool-fs`, que renderiza el texto y los errores del provider como resultados de herramienta de sistema de archivos acotados y retenidos.

#### Efecto en la caché KV

Sin invalidación directa; el Consumer designado es propietario de cualquier cambio de prefijo de solicitud.

## Limitaciones conocidas y trabajo diferido

- **Mutaciones solo de texto por contrato** — las lecturas de texto y ambas mutaciones rechazan contenido binario/no-UTF-8 con `FS_NOT_TEXT`; `readBytes` es la única primitiva de bytes en bruto, y las mutaciones seguras para binarios siguen siendo un aplazamiento deliberado de [el Agent Note de schemas de herramientas](../../../.agents/notes/implemented/feature/2026-06-17-filesystem-tool-schemas.es.md).
- **Solo doce primitivas** — sin borrado, renombrado/movimiento, copia ni vigilancia; `listDir` es de un solo nivel, con la recursión, el globbing, la paginación y la búsqueda fuera de alcance según [el Agent Note de listado de directorios](../../../.agents/notes/archived/architecture/2026-07-03-filesystem-directory-listing-seam.md).
- **Sin plazo de E/S** — el seam no arma ningún tiempo de espera; la cancelación es un `AbortSignal` opcional de mejor esfuerzo por primitiva (la postura deliberada de [la familia fs](../README.es.md)).
- **Resolver-y-luego-operar cuesta a un backend remoto dos idas y vueltas por llamada de herramienta** — plegar o cachear la resolución se deja a ese backend.
