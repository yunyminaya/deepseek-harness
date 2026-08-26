# @deepseek-ai/dsh-session-persistence-jsonl

[English](README.md) | Español

El backend JSONL de persistencia de sesión duradera — un `SessionPersistence` concreto (el seam `dsh-session-persistence`). Cada sesión tiene un log JSONL lógico de solo anexión, almacenado como `.jsonl.zstd` por defecto o como `.jsonl` crudo cuando la compresión está deshabilitada.

## Diseño en disco

```
<root>/
  --<normalized-cwd>--/          # readable project directory (or _no-cwd/)
    <encoded-id>/                # session-owned directory
      session.jsonl.zstd         # default: checksummed header frame + append frames
      session.jsonl              # only with compression: 'none'
```

- La primera línea lógica es el `SessionHeader` inmutable etiquetado `{ type: 'session', version, id, cwd?, createdAt, parentSession?, seedLength?, origin?, delegationDepth, agentPreset? }`. `delegationDepth` es obligatorio en disco y es `0` para una sesión de nivel superior; un valor ausente o inválido rechaza el log. `agentPreset` es duradero porque decide las herramientas y el prompt de la sesión reanudada — restaurar una composición distinta reproduciría un historial sobre el que el modelo ya no puede actuar. Cada línea lógica posterior es un registro de almacenamiento; los eventos `assistant/chunk` nunca se descartan y `seq` permanece contiguo en todo el log decodificado (`events[i].seq === i`).
- Un registro de almacenamiento es un `SessionEvent` JSON verbatim, o — para una secuencia elegible cuando `packChunks` está habilitado — una **fila de fragmentos empaquetada** (`text-chunks` / `reasoning-chunks` / `tool-call-chunks`; etiquetas simples sin barra como el `session` de la cabecera, de modo que las etiquetas de fila no pueden confundirse con tipos de evento): una línea que contiene una secuencia de ≥3 eventos delta `assistant/chunk` consecutivos del mismo bloque, con `seq0`/`time0` más las diferencias `dt` por miembro que reconstruyen exactamente el `seq`/`time` de cada miembro. El códec sin pérdidas vive en `@deepseek-ai/dsh-session` (`packChunkRuns`/`decodeStorageRecord`) e incluye en la whitelist formas exactas — cualquier cosa no reconocida se almacena verbatim. La lectura es independiente del diseño: `load` decodifica siempre las filas, de modo que los archivos empaquetados, no empaquetados y mixtos se cargan de forma idéntica.
- El directorio de proyecto mantiene el cwd normalizado legible para la navegación y está acotado por los límites de componentes del sistema de archivos. El reemplazo de separadores y el truncado son deliberadamente con pérdidas, de modo que las cadenas cwd que se normalizan igual comparten un directorio de proyecto; los ids de sesión siguen seleccionando directorios de sesión distintos. En un sistema de archivos insensible a mayúsculas, la validación de identidad acepta una grafía alternativa de ruta solo cuando la canonicalización del sistema de archivos resuelve ambas grafías al mismo transcript. La raíz configurada permanece bajo control del despliegue: puede ser local al proyecto, compartida, temporal o centralizada. La [decisión de directorios de sesión de proyecto](../../../.agents/notes/implemented/architecture/2026-07-24-project-session-directories.md) registra esta compensación.
- Los ids de sesión son cadenas marcadas sin validar, de modo que se escapan de forma inyectiva a un único segmento de ruta seguro antes de usarse (sin recorridos ni colisiones). El directorio resultante queda reservado para artefactos adicionales propiedad de la sesión; el descubrimiento solo lee el nombre de archivo fijo del transcript.

## Configuración

| Clave | Tipo | Notas |
|---|---|---|
| `root` | `string` (obligatorio) | Directorio raíz de todos los archivos de sesión. **Sin valor por defecto** — un valor por defecto de `process.cwd()` dispersaría los archivos a medida que cambia el cwd del proceso (llamadas bash, subprocesos). Una raíz existente debe ser un directorio legible; una raíz ausente se crea en la primera materialización. |
| `packChunks` | `boolean` (por defecto `true`) | Escribe las secuencias elegibles de fragmentos delta como filas empaquetadas (logs lógicos ~60% más pequeños medidos en una sesión de programación real). Pon `false` para diagnósticos de un evento por línea; la lectura de filas empaquetadas funciona independientemente de este conmutador del lado de escritura. |
| `compression` | `'zstd' \| 'none'` | Tiene por defecto `'zstd'`; `'none'` conserva texto UTF-8 delimitado por saltos de línea. |
| `preparedSessionCacheSize` | entero positivo (por defecto `5`) | Máximo de sesiones no publicadas retenidas tras la inspección de historial en frío para su reutilización por el resume. |
| `writeBatchMaxDelayMs` | entero positivo (por defecto `200`) | Ventana fija de coalescencia después de que una cola de eventos en vivo inactiva reciba trabajo. Los eventos posteriores no la reinician; flush y teardown la omiten. No acota la latencia del event loop, de las operaciones serializadas ni del backend. Como máximo el límite de temporizador de Node de `2_147_483_647` ms. |

`locate(meta)` devuelve `{ kind: 'jsonl', path }` para el transcript fijo dentro de los directorios de proyecto/sesión resueltos. No realiza E/S del sistema de archivos: el objetivo puede devolverse antes de que existan el directorio o el archivo, y un archivo existente contiene solo el último prefijo volcado (flushed).

## Codificación física

El artefacto por defecto es una concatenación estándar de [marcos Zstandard](../../../.agents/notes/implemented/architecture/2026-07-19-zstandard-jsonl-session-logs.md) independientes: un marco con checksum que contiene solo la línea de cabecera, seguido de un marco con checksum por lote de anexión duradero. El backend usa la API Zstandard integrada de Node con su nivel de compresión por defecto y no expone ningún control de nivel. El listado lee y valida solo el marco de cabecera. `compression: 'none'` conserva las mismas líneas lógicas en la representación cruda original.

Una raíz pertenece a una única codificación. El descubrimiento de arranque y la búsqueda dirigida rechazan el sufijo opuesto con un error que nombra el artefacto incompatible e instruye al llamante a seleccionar el modo coincidente o una raíz separada. Los artefactos planos `<project>/<id>.jsonl*` también se rechazan en lugar de ignorarse. No hay migración, ni fallback de raíz mixta, ni escritura dual.

## Durabilidad y semántica de fallos

- **Identidad de almacenamiento acotada.** La búsqueda exige un directorio de sesión coincidente entre los directorios de proyecto legibles, y después verifica que el id de la cabecera sea igual al id solicitado y que el id/cwd de la cabecera derive la ruta del transcript seleccionado. El listado aplica la misma comprobación de ruta y rechaza ids duplicados. Los fallos de identidad ocurren antes de la reparación o la anexión.
- **Materialización perezosa.** `create(meta)` no escribe nada; en la primera `append`, el backend escribe y hace `fsync` de la cabecera codificada y del primer lote en un archivo temporal. POSIX lo publica sin sobrescritura mediante un hard link y hace `fsync` del directorio padre. Windows lo publica sin sobrescritura mediante `MoveFileExW(..., MOVEFILE_WRITE_THROUGH)` y crea los directorios ausentes con el mismo patrón write-through. Una sesión creada pero nunca anexada no deja nada en disco y está ausente de `list`.
- **Solo anexión.** Los eventos volcados nunca se reescriben. Los lotes crudos posteriores anexan líneas; los lotes comprimidos anexan un marco. Ambas vías hacen `fsync`, y un fallo de escritura o de sync detectado revierte el archivo a su longitud previa en bytes.
- **Recuperación de fallos — conserva el trabajo válido de la cola.** `load` valida cada marco comprimido completo y escanea su JSONL descomprimido. Si el último marco es estructuralmente incompleto, el lector conserva sus registros decodificados completos, trunca desde el inicio de ese marco y recodifica esos registros con los cierres sintéticos de herramienta, paso y turno que exige el [contrato de persistencia](../../../.agents/notes/implemented/architecture/2026-06-14-session-persistence.md) compartido. El modo crudo trunca desde su primera línea incompleta. Un artefacto comprimido existente sin marco de cabecera completo, un fallo de checksum/descompresión en un marco completo o un defecto en o antes del último `turn/end` confirmado es corrupción y se rechaza.
- **Inspección no mutante.** `inspect()` devuelve una vista lógica equilibrada e inmutable y puede sintetizar cierres de recuperación en memoria, sin truncar una cola incompleta ni cambiar la revisión ligera.
- **Seq contiguo.** `append` rechaza un lote cuyo primer `seq` no continúa el log almacenado, y rechaza un `event.data` no serializable en JSON nombrando el tipo de evento infractor.
- **Revisiones ligeras.** `listSnapshots(signal?)` identifica un log por su dispositivo, inodo, tamaño y marcas de tiempo en nanosegundos, evitando un análisis completo del log y cambiando después de anexiones, reparaciones, reemplazos o cambios del store. Una lectura de prefijo completo exige la misma identidad antes y después de leer los bytes, y `readStoredRevision()` usa esa identidad para validar las preparaciones retenidas sin cargar el log. El listado de instantáneas reenvía la señal exacta a través del descubrimiento de artefactos y comprueba la cancelación alrededor de cada `stat`; como el `stat` del sistema de archivos no es interrumpible, la cancelación espera a que se asiente la llamada activa y después rechaza sin iniciar otra.

## Vía de escritura

El plugin copia los eventos de sesión congelados en un controlador por sesión viva. El primer evento pendiente inicia la ventana fija de batching configurada, y los eventos posteriores se unen sin reiniciarla. La caducidad inicia una anexión duradera; los eventos admitidos durante esa escritura forman un lote de seguimiento acotado por separado. `session/flush` cancela la espera y drena los lotes actuales y pendientes. Un cursor por sesión impide que las sesiones reanudadas vuelvan a anexar eventos almacenados, y las sesiones vivas se siembran cuando se carga el plugin. La instancia de backend propietaria serializa las operaciones de una sesión; el disposal drena cada controlador retenido antes del teardown. Cada evento lógico permanece presente: el batching solo permite que un marco comprimido o un fsync crudo transporten más registros.

## Experiencia de modelo

### Historial de conversación reanudado

#### Lo que ve el modelo

El almacenamiento JSONL no contribuye prompt ni schema en vivo. La carga restaura el historial de surface almacenado y conserva las cabeceras de solicitud previas para la reconstrucción; el nuevo loop compone su envoltorio actual. La recuperación equilibra una solicitud de asistente sin llamada duradera con `TOOL_NOT_STARTED`; una llamada duradera sin resultado se convierte en `TOOL_OUTCOME_UNKNOWN`, que dice al modelo que reintente solo el trabajo de solo lectura o idempotente y que verifique los posibles efectos secundarios o pregunte al usuario. Los registros crudos `assistant/chunk` no duplican mensajes.

#### Efecto en tokens

Cero tokens de solicitud en vivo. Un agent reanudado paga por el historial retenido y su envoltorio actual, más el resultado de reparación citado por cada llamada interrumpida.

#### Efecto en la caché KV

El almacenamiento JSONL no muta los prefijos de solicitud en vivo. Un loop reanudado solo puede reutilizar la caché del provider cuando su historial reconstruido, su envoltorio actual y la ruta del modelo coinciden; los resultados de reparación de fallos se anexan.

## Limitaciones conocidas y trabajo diferido

- **Solo se carga la codificación configurada y el `SESSION_FORMAT_VERSION` actual (v0)** — cambiar la compresión exige una raíz separada/nueva o seleccionar el modo crudo heredado; el formato de pre-release no tiene migración.
- **El diseño de almacenamiento de archivo plano no se carga** — usa una raíz separada o mueve los artefactos de pre-release al diseño de directorios de proyecto/sesión antes de cargar.
- **Los archivos comprimidos no son legibles por línea directamente** — usa el backend para cargarlos, o selecciona `compression: 'none'` antes de escribir una raíz nueva cuando se necesiten lectores de líneas externos.
- **Nada borra los archivos de sesión** — los logs se acumulan bajo `root` hasta que se eliminan externamente (el seam no tiene API de borrado).
- **Un escritor en vivo por sesión** — la anexión y la reparación se coordinan solo dentro de la instancia de backend propietaria. Otra instancia de backend o proceso no debe escribir la misma sesión hasta que ese propietario alcance un disposal en quiescence; la publicación inicial del mismo id sigue siendo segura frente a colisiones gracias al hard link POSIX sin sobrescritura o al rename write-through de Windows sin reemplazo.
- **La materialización POSIX exige soporte de hard links** — la primera anexión usa `link()` para que las carreras del mismo id fallen en lugar de sobrescribir un log confirmado; Windows usa rename write-through sin reemplazo.
