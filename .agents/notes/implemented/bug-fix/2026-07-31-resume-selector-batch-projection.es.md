# Agent Note: El selector de resume solo pliega títulos

Status: implemented

[English](2026-07-31-resume-selector-batch-projection.md) | [中文](2026-07-31-resume-selector-batch-projection.zh.md) | Español

## Problema

Abrir el selector `/resume` de la TUI llamaba a `sessionQuery.readSession()` una vez por cada sesión listada bajo un `Promise.all` sin límite. Cada llamada volvía a listar todo el almacén de persistencia dentro de `SessionCorpus.load()` (listados O(N²)), leía y descomprimía el registro completo, validaba cada evento por reproducción a través del constructor `Session` y clonaba en profundidad la cabecera y los eventos hasta tres veces — todo para derivar el título de una fila del selector, la hora de última actividad, la etiqueta del último `turn/end`, la ruta provider/modelo y la fase de objetivo. En un almacén real (185 sesiones, 87 MB comprimidos, ~353k eventos) el selector tardaba decenas de segundos en abrir, y el costo crecía con el tamaño total de los registros y no con el número de sesiones.

## Decisión

Las filas del selector no pliegan nada más que títulos, y todo lo demás que muestra una fila proviene de metadatos:

- Los títulos provienen del sistema de proyección: `session-title` ya registra una unidad `title`, así que una fila viva lee la instantánea del registro, una fila persistida lee la fila del checkpoint durable (`sessionProjectionCache.cachedSnapshot`, cero I/O) y solo una fila sin un checkpoint utilizable paga un `coldSnapshot` — checkpoint más una cola `readFrom`, reescrita para que el siguiente escaneo sea de cero I/O. Las lecturas en frío están limitadas por la configuración TUI `resumeScanConcurrency`. Una composición sin la caché recurre a un lote acotado `readTitleSnapshots` sobre los registros; cualquiera de las dos vías aísla una falla por fila en el fallback deshabilitado «Unreadable session».
- La marca de tiempo de actividad nunca lee un registro: una sesión viva usa la hora de su último evento en memoria; una sesión persistida aplica stat al artefacto nombrado por el `sessionPersistence.locate()` opcional (mtime), recurriendo a la hora de creación de la cabecera cuando el backend no localiza un artefacto por sesión (SQLite) o el stat falla. Cualquier adición mueve el mtime, así que un mero límite de recogida ahora flota hacia arriba una sesión navegada — aceptado como el precio de una marca de tiempo solo de metadatos.
- La etiqueta del último turno, la columna de ruta provider/modelo y la fase de objetivo desaparecen de las filas. La disponibilidad de la ruta ahora la aplica el preflight al pulsar Enter, que lee y valida por reproducción el único registro elegido a través de `readSession` antes de la entrega.

La capa del selector se abre de forma síncrona cuando `/resume` se despacha, antes de que el escaneo termine: un conjunto candidato `undefined` renderiza un marcador «Loading sessions…», el selector posee la entrada de la terminal desde su primer fotograma, Enter informa que las sesiones siguen cargando y Escape cancela. Cerrar la capa aborta el escaneo a través del `AbortSignal` que aceptan los métodos de consulta; la resolución tardía de un backend que ignora la señal se descarta mediante una comprobación de obsolescencia. El escaneo terminado intercambia las filas mediante `setCandidates` (limpiando un error obsoleto de carga en curso) sin reemplazar la capa; una activación en cola detrás de una predecesora en cierre recibe un conjunto ya escaneado en su construcción; un único `catch` abarca el listado, los títulos y los mtimes, de modo que cualquier falla del escaneo cierra la capa y reporta un aviso en lugar de dejar varado el marcador de carga.

Ninguna superficie de session-query o session-persistence cambió. La composición TUI publicada gana el registro de proyecciones, el almacenamiento y las filas de la caché de proyecciones (reflejando la capa web sobre la misma raíz `storages`, así que los checkpoints escritos por cualquiera de las dos superficies sirven a ambas); el primer escaneo sobre un almacén preexistente todavía lee cada registro una vez para sembrar checkpoints, y cada escaneo posterior es solo de metadatos.

## Alternativas consideradas

**Mantener las columnas de ruta/turno/objetivo por fila mediante una proyección genérica por lotes (`projectSessions`).** Implementada primero y luego rechazada: seguía descomprimiendo y analizando cada registro en cada `/resume`, así que el costo de navegación seguía en O(bytes totales de registro), y engordaba la API pública de session-query por un solo consumidor. El contrato público se revirtió; `readTitleSnapshots` sigue usando el `projectMany` interno sin cambios.

**Arreglar solo el listado O(N²) dentro de `SessionCorpus.load()`.** Rechazado como corrección principal: la descompresión completa por candidato, la validación por reproducción y el triple clonado dominaban en registros grandes. El prelistado redundante en `load()` sigue siendo una limpieza candidata con implicaciones de semántica de error.

**Exponer una hora de última modificación a través de `listSnapshots`/`SessionRecord`.** Lo más limpio respecto al seam, pero toca el contrato de persistencia, ambos backends y la forma del registro de consulta por lo que la TUI ya puede derivar de `locate()` más un stat. Reintroducir si un segundo consumidor necesita horas de actividad por metadatos.

**Un índice de títulos persistido a medida o una caché de títulos local de la TUI.** Rechazado: la caché de session-projection ya es el sistema de checkpoints durables propietario con un contrato de invalidación (`stateVersion`, vinculación de identidad, anclaje a registro reducido); montarla supera añadir una caché paralela.

## Consecuencias

Abrir `/resume` ejecuta un listado, un stat por fila persistida y lecturas de títulos por fila que tocan solo filas de checkpoint y colas de registro una vez que existen los checkpoints — metadatos O(número de sesiones) en lugar de O(bytes totales de registro); la vía de fallback sin caché sigue siendo una única pasada de títulos acotada. Las filas muestran solo título, marca de tiempo, estado e id; los problemas de ruta afloran como un error de preflight al pulsar Enter en vez de una fila deshabilitada, y una sesión que falla la reproducción la captura el preflight y no el listado. Las sesiones navegadas y luego abandonadas flotan hacia arriba por su mtime de recogida. Los servicios `sessionQuery` falsos en las pruebas TUI proveen `readTitleSnapshots` junto con `listSessions`/`readSession`, y el arnés de pruebas reenvía un `locate` opcional. Como el selector toma el foco de inmediato, iniciar un segundo escaneo exige cerrar primero la capa actual — un segundo `/resume` tecleado durante un escaneo cae en el campo de búsqueda, que es la captura de entrada prevista.
