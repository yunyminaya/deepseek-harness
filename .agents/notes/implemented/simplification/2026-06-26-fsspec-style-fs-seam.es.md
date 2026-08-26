# Agent Note: Dividir el seam del sistema de archivos — mutaciones de texto del provider más el plugin `dsh-fs-observation-policy`

Status: implemented

[English](2026-06-26-fsspec-style-fs-seam.md) | Español

## Problema

La capacidad de sistema de archivos del [seam de capacidad de sistema de archivos](../architecture/2026-06-17-filesystem-capability-seam.es.md) hace que hoy un único servicio abstracto `FileSystem` sea propietario de dos trabajos distintos:

1. **Operaciones de provider** — resolución de destinos, metadatos stat/version, lecturas/flujos de texto, escrituras atómicas y ediciones literales con guardia.
2. **Política orientada al agente** — ventanas de línea, semántica de edición literal y estado observado de leer-antes-de-escribir/editar.

Eso obliga a cada backend futuro a reimplementar la semántica de lectura orientada al modelo y la política de observación. `readPage` devuelve líneas numeradas y metadatos de vista; el servicio base almacena el estado del archivo por propietario y distingue las lecturas `full` de las `partial`. Son políticas útiles, pero no son primitivas de provider de sistema de archivos. La mutación literal de texto es distinta: la guardia de versión, la coincidencia literal, la detección de ambigüedad y la reescritura atómica deben permanecer juntas dentro del límite de mutación del provider, pero el nombre actual `applyEdit` y el seam circundante atan esa operación de provider a la antigua forma de política de leer-antes-de-editar.

Esto también crea un callejón sin salida de UX real: una lectura con ventana registra `view: partial`, y las vistas parciales no pueden autorizar `edit`. Por tanto, un modelo que lee las líneas 100-150 de un archivo grande no puede editar la línea 120 a menos que primero consiga una lectura `full`, que puede ser imposible para un archivo que supera el tope de lectura. La edición literal solo necesita frescura: los bytes que se van a hacer coincidir deben seguir siendo de la versión que el modelo leyó.

El antiguo Agent Note ya había diferido un paquete `@deepseek-ai/dsh-fs-observation-policy` separado. Esta decisión construye esa capa y mantiene `ctx.fs` cerca de las primitivas de almacenamiento estilo fsspec (`info`/`cat`/`open`), sin convertirlo en fsspec completo.

## Decisión

Dividir la pila en cuatro capas:

```text
tool          dsh-tool-fs       model-facing schemas + read windowing + text rendering; the EXECUTOR (reads/writes/edits via ctx.fs, dispatches the fs/* events)
policy        dsh-fs-observation-policy  observed-state + read-before-edit + write/edit freshness, contributed through the fs/* event gate (no service)
provider contract dsh-fs            ctx.fs: text IO + atomic mutation primitives (optional version guard)
provider      dsh-fs-local      local implementation of ctx.fs
```

`dsh-tool-fs` conserva los mismos schemas `read`/`write`/`edit` orientados al modelo. Es el ejecutor: inyecta `fs` (no un servicio de política) y accede a `ctx.fs` directamente, es propietario del ventaneado de lectura y despacha los eventos `fs/*` para que `dsh-fs-observation-policy` pueda aplicar la puerta y registrar.

Este Agent Note decidió la división en cuatro capas, el contrato de provider y la política de frescura. El ACOPLAMIENTO herramienta↔política se refinó después en el [Agent Note de puerta de eventos](../architecture/2026-06-26-file-context-as-event-gate.es.md): `dsh-fs-observation-policy` es un plugin de PUERTA que participa a través de los eventos `fs/*` en lugar de un servicio de método `ctx.fileContext`, de modo que la herramienta no está acoplada por método a él y el ventaneado de lectura + la E/S de fs viven en `dsh-tool-fs`. Este documento describe esa forma de puerta de eventos ya aterrizada; la guardia de versión del provider es opcional (omitir = provider puro incondicional).

## Contrato de provider

`@deepseek-ai/dsh-fs` se reduce a E/S de texto de provider más mutación de texto con guardia:

```ts ignore-check
abstract resolve(path: string, opts?: { cwd?: string; signal?: AbortSignal }): Promise<FsTarget>
abstract stat(target: FsTarget, signal?: AbortSignal): Promise<FsInfo | undefined>
abstract readText(target: FsTarget, signal?: AbortSignal): Promise<string>
abstract streamText(target: FsTarget, signal?: AbortSignal): Promise<AsyncIterable<string>>
abstract writeText(target: FsTarget, content: string, expected: FsWriteIntent, signal?: AbortSignal): Promise<FsWriteOutcome>
abstract editText(target: FsTarget, edit: FsEditRequest, expected: { version: FsVersion }, signal?: AbortSignal): Promise<FsEditOutcome>

interface FsInfo {
  version: FsVersion
  type: 'file' | 'directory' | 'other'
  size?: number
}

type FsWriteIntent =
  | { kind: 'createIfAbsent' }
  | { kind: 'replaceIfVersion'; version: FsVersion }
```

`stat` devuelve metadatos, no contenido. `version` es el token de frescura; `type` permite al ejecutor rechazar directorios/archivos especiales antes de leer; `size` permite a la herramienta `read` elegir entre `readText` y `streamText` sin sondear por fallo. `undefined` significa ausente.

`readText` lee el archivo de texto regular completo. `streamText` transmite la misma semántica de texto para archivos grandes. Ambas primitivas de provider son propietarias de las comprobaciones de archivo regular, la decodificación UTF-8, el rechazo binario/NUL y `FS_NOT_TEXT`; la capa de política nunca maneja bytes crudos ni reimplementa la decodificación entre fragmentos. `readText` es la primitiva de archivo pequeño/completo directa, mientras que las lecturas grandes orientadas al modelo usan `streamText`.

`writeText` es archivo temporal atómico + rename con una expectativa de escritura explícita. `createIfAbsent` crea un destino ausente y rechaza un destino existente con `FS_NOT_OBSERVED`; es la vía usada cuando el propietario no tiene ninguna lectura previa. `replaceIfVersion` reemplaza solo cuando el destino existe en la versión observada; un destino ausente o una discrepancia de versión lanza `FS_STALE_VERSION`.

`editText` es una mutación de texto con guardia a nivel de provider. Cuando tiene guardia, primero verifica que el destino sigue existiendo en `expected.version`, luego lee el texto actual, aplica el reemplazo literal y escribe atómicamente. La comprobación de obsolescencia debe ocurrir antes de la coincidencia literal para que una edición basada en una lectura antigua informe `FS_STALE_VERSION`, no `FS_EDIT_NOT_FOUND` o `FS_AMBIGUOUS_EDIT` por coincidir contra contenido más nuevo. Conservar esta primitiva en el contrato de provider preserva el bloqueo local al backend y permite que un futuro backend remoto implemente compare-and-edit nativo sin obligar a la capa de política a arrastrar el archivo completo a través de él.

Este es un seam de *almacenamiento de texto*, deliberadamente medio nivel por encima del fsspec a nivel de bytes (`cat`/`open` devuelven bytes crudos). La decodificación UTF-8, el rechazo binario/NUL, las escrituras de archivo completo con guardia y las ediciones literales de texto con guardia viven en el provider para que la capa de política nunca toque bytes crudos, reimplemente la decodificación entre fragmentos ni separe las comprobaciones de obsolescencia de la sección crítica de la mutación. Los conceptos orientados al modelo siguen fuera del provider: no se filtran ventanas de línea, líneas numeradas, pies de página renderizados ni el almacén de estado observado.

Eliminados de `dsh-fs`: `readPage`, `FsExpectation`, `FsView`, `FsStateSource`, `FsReadRequest`, `FsTextLine`, las constantes de línea/ventana, `formatReadBody` y el `WeakMap` de estado observado. `applyEdit` se reemplaza por la primitiva de provider más estrecha `editText`, cuyo contrato es la mutación literal de texto con guardia de versión en lugar de la autorización de lectura de la capa de política. El código `FS_PARTIAL_OBSERVATION` también sale de la taxonomía `FsErrorCode`: la autorización por frescura no tiene distinción parcial/completo, así que nada puede lanzarlo. `FsTargetKey` y `FsVersion` pasan a ser ids opacos con marca bajo el [Agent Note de ids con marca](../architecture/2026-06-20-branded-ids.es.md).

## Contrato de política

`@deepseek-ai/dsh-fs-observation-policy` es un plugin, no un servicio: no registra ninguna clave `ctx.*` ni inyecta nada. Es propietario de la política de frescura de escritura/edición y del estado observado que no pertenecen a la clase base del provider `FileSystem` (donde un backend en sandbox o remoto heredaría de otro modo una política de observación orientada al modelo que no le corresponde). Contribuye esa política a través de la puerta de eventos `fs/*` que despacha el ejecutor.

El estado observado vive aquí como `WeakMap<owner, Map<targetKey, FsVersion>>`. Una entrada existe si y solo si el propietario ha leído, escrito O editado ese destino (cada éxito emite `fs/observed`), así que su presencia *es* el registro de observación previa — no hay un flag `hasRead` separado. El propietario se deriva estructuralmente del actor de evento opaco (`{ agent?: { session? } }`), una forma que vive en `dsh-fs-observation-policy`, no en `dsh-fs`.

El plugin decide tres eventos `fs/*`:

- `fs/write-intent` — sin observación previa ⇒ `{ kind: 'createIfAbsent' }` (solo los archivos nuevos pueden crearse a ciegas); con observación previa ⇒ `{ kind: 'replaceIfVersion', version: vObserved }` (los archivos existentes solo se reemplazan si no han cambiado desde la observación). Decisión de un solo slot; no llama a `next()`.
- `fs/edit-intent` — exige una observación previa del propietario (si no, `FS_NOT_OBSERVED`); devuelve `{ version: vObserved }` como base del CAS. No implementa el reemplazo literal — autoriza y suministra la versión, y la sección crítica de mutación del provider aplica la guardia, de modo que las ediciones concurrentes basadas en la misma versión observada siguen siendo uno-gana/uno-obsolescente.
- `fs/observed` — registra `{ version }` para este propietario+destino tras una lectura/escritura/edición exitosa. `WeakMap.set` síncrono, solo con efectos secundarios.

El plugin NO hace ninguna E/S de sistema de archivos: «¿has observado este archivo?» es una búsqueda en el `WeakMap`, y «¿la versión que leíste sigue siendo actual?» se decide dentro de `ctx.fs.editText`/`writeText` en el mismo bloqueo atómico que realiza la mutación — el plugin solo suministra `vObserved` como base.

## Contrato de herramienta

`dsh-tool-fs` conserva los mismos schemas y la entrada de prompt. `read` sigue exponiendo `file_path`, `offset` y `limit`; `write` y `edit` no cambian. Es el ejecutor: valida los argumentos del modelo, lee/escribe/edita a través de `ctx.fs` directamente, es propietario del ventaneado de líneas y del renderizado de resultados (`N: text`, pie de página, envoltorio `<path>/<content>`), y despacha los eventos `fs/*`.

Cada mutación despacha su cascada de intención con un default `undefined` de provider puro, luego llama a `ctx.fs`, y luego emite `fs/observed`: p. ej. `write` hace `ctx.waterfall('fs/write-intent', target, exec, () => undefined)` → `ctx.fs.writeText(target, content, intent)` → `ctx.emit('fs/observed', …)`. Un `read` hace stat una vez, lee/transmite, construye la ventana y emite `fs/observed`. Pasar `exec` como actor permite a `dsh-fs-observation-policy` derivar el propietario sin que la herramienta llegue a la política.

Como la política se contribuye a través de eventos con un default `undefined`, `dsh-tool-fs` no está acoplada por método a `dsh-fs-observation-policy`: con el plugin ausente, cada cascada de intención cae hasta `undefined` (escritura/edición incondicional de provider puro) y `fs/observed` no tiene listener. Cargar el plugin de nuevo superpone la política de leer-antes-de-escribir/editar.

## Límite de concurrencia

Las actualizaciones en proceso son seguras: el backend local conserva el bloqueo de mutación por destino existente, de modo que verificar-versión-luego-rename se serializa y una actualización perdedora ve `FS_STALE_VERSION`.

Las creaciones en proceso están protegidas por el mismo bloqueo de mutación por destino: dos llamadores compitiendo con `createIfAbsent` se serializan, uno crea, y el siguiente ve que el destino existe y recibe `FS_NOT_OBSERVED`. Las creaciones entre procesos son solo de mejor esfuerzo; una guardia local de stat-luego-rename no puede dar garantías portables de exclusividad de creación en todos los backends futuros.

Las escrituras entre procesos son frescura de mejor esfuerzo más reemplazo atómico: `mtime:size` normalmente detecta los guardados del editor, pero las escrituras del mismo tick y mismo tamaño pueden fallar; el temp+rename atómico evita archivos rasgados pero no todas las actualizaciones perdidas.

## Reemplaza

Este Agent Note revierte dos decisiones del [seam de capacidad de sistema de archivos](../architecture/2026-06-17-filesystem-capability-seam.es.md) y acota una tercera:

- La política de leer-antes-de-escribir/editar sale de `ctx.fs` y entra en el plugin `dsh-fs-observation-policy` (en la puerta de eventos `fs/*`).
- Las lecturas de texto ya no devuelven registros de línea numerados por el backend ni vistas `full`/`partial`; la autorización se basa en la frescura de versión, así que una lectura con ventana puede autorizar la edición cuando el archivo no ha cambiado.
- La edición literal ya no está detrás de la antigua API `applyEdit` que mezclaba la mutación del backend con la política de observación del seam. Sigue siendo una primitiva de provider como `editText`, porque la guardia de versión + la coincidencia literal + la reescritura atómica deben permanecer dentro de la sección crítica de mutación del provider.

Conserva la disciplina de Service Definition / Service Provider / Consumer, la regla de que el consumidor nunca importa el backend, los metadatos de destino/versión/visualización definidos por el backend, las escrituras locales atómicas y la taxonomía compartida `FsError`.

## Verificación

`dsh-fs` expone exactamente `resolve`/`stat`/`readText`/`streamText`/`writeText`/`editText` (`stat` devuelve `FsInfo | undefined`, `writeText` toma `FsWriteIntent`), con los tipos/primitivas eliminados fuera; `dsh-fs-local` no lleva lógica de línea, vista ni `formatReadBody`; los schemas orientados al modelo permanecieron byte a byte sin cambios. Las pruebas fijan que una lectura con ventana autoriza una edición posterior de un archivo sin cambios, que una edición basada en una lectura obsoleta informa `FS_STALE_VERSION` antes de intentar la coincidencia literal, que el comportamiento CAS de versión se preserva y que el contrato de observación se cumple (una lectura de la herramienta `read` registra estado observado; una lectura directa de `ctx.fs` no); `dsh-fs-observation-policy` tiene cobertura de HMR/destrucción.

## Extensión posterior

El seam se extendió después con la lista de directorios directa mediante [Añadir listado de directorios directo al seam del sistema de archivos](../../archived/architecture/2026-07-03-filesystem-directory-listing-seam.md). Ese seguimiento se registra por separado para que esta nota siga describiendo el reajuste estilo fsspec que se publicó originalmente.

## Alternativas consideradas

- **fsspec a nivel de bytes (`cat`/`open` devolviendo bytes crudos)** — rechazado: el seam es deliberadamente de almacenamiento de texto, medio nivel por encima, para que la decodificación UTF-8, el rechazo binario/NUL y las mutaciones de texto con guardia vivan una vez en el provider y la capa de política nunca toque bytes crudos ni separe las comprobaciones de obsolescencia de la sección crítica de la mutación.
- **Un servicio de método `ctx.fileContext` concreto** — la forma de política original de este Agent Note; rehecho por el [Agent Note de puerta de eventos](../architecture/2026-06-26-file-context-as-event-gate.es.md) en el plugin de puerta, de modo que la herramienta nunca está acoplada por método a la política.
- **Conservar `readPage` y la autorización de vista `full`/`partial` en el provider** — la forma previa al reajuste que revierte la sección Reemplaza: la completitud de la vista no es lo que la seguridad de edición necesita, sino la frescura de versión, y la regla de vista hacía imposible editar los archivos grandes que superan el tope de lectura.

## Consecuencias

- Añade un cuarto paquete de fs y una nueva capa de plugin. Es intencional: es la capa de política antes diferida, no un segundo contrato abstracto de backend.
- El uso directo de `ctx.fs` se salta la política: un `ctx.fs.readText` directo no emite `fs/observed`, así que bajo la política por defecto una `edit` posterior se rechaza con `FS_NOT_OBSERVED` hasta que el archivo se lea a través de la herramienta `read`. El fallo es explícito y está documentado.
- El ventaneado de líneas de archivos grandes pasa del backend a la herramienta `read` en `dsh-tool-fs`; la decodificación de texto y el rechazo binario permanecen en `ctx.fs.streamText`, así que esto es solo reubicación del ventaneado, no una segunda implementación de E/S de texto.
- Conservar `editText` en el contrato de provider significa que cada backend debe implementar el contrato de reemplazo literal. Es intencional: la operación no es almacenamiento puro, pero guardia obsoleta + coincidencia literal + reescritura atómica es la unidad que debe permanecer junta para una atribución de errores y un comportamiento de concurrencia correctos. El contrato debería seguir siendo estrecho y solo de texto para que los backends futuros puedan implementarlo de forma nativa o mediante reescritura de archivo completo.
- La frescura permite `write` de archivo completo tras una lectura con ventana. Es más débil que la antigua comprobación de vista, pero evita que los archivos grandes sean imposibles de editar; la guía de prompts sigue desaconsejando los reemplazos completos a ciegas.
