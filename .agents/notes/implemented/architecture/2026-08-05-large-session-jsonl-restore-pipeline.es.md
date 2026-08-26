# Agent Note: Pipeline de restauración JSONL para sesiones grandes

Status: implemented

[English](2026-08-05-large-session-jsonl-restore-pipeline.md) | Español

## Problema

Restaurar una sesión almacenada la activa y materializa su log de eventos autoritativo completo antes de que el agent pueda ejecutarse. Los artefactos JSONL grandes hacían que esa operación puntual pagara varios costes evitables: cada frame Zstandard independiente creaba y cerraba un contexto de decodificador, el texto plano decodificado se acumulaba y se reescaneaba como buffers y cadenas de todo el log, y los eventos recién analizados pasaban por caminos genéricos de instantánea y congelación profunda diseñados para valores prestados o cíclicos.

Un profile representativo contenía 61.8 MiB de datos Zstandard, 97.1 MiB de texto plano y 1,307,073 eventos. El camino de restauración debe reducir su coste de CPU y memoria sin debilitar la validación de checksums, la detección de corrupción de la región confirmada, la recuperación de colas cortadas, la validación de secuencia y superficie, ni la inmutabilidad del log de sesión.

## Decisión

La restauración es un único pipeline de transferencia de propiedad desde el artefacto de persistencia hacia `Session.fromRestore`. El artefacto comprimido sigue siendo el buffer fuente, mientras que cada etapa de decodificación y escaneo consume la salida de la etapa anterior de forma incremental sin retener una copia de texto plano o analizada de todo el log; el array de eventos resultante es la única representación decodificada completa.

### Decodificación de frames

El escáner estructural Zstandard identifica los rangos de frames completos antes de decodificar. El primer frame dedicado se decodifica y analiza por separado como cabecera de sesión; los frames de texto plano posteriores se emiten en orden hacia el escáner JSONL.

`ZstdFrameDecoder` da al lector un único ciclo de vida para implementaciones síncronas intercambiables. La implementación preferida sondea la forma de stream soportada de Node 22, 24 y 26, reutiliza un contexto privado de decodificador nativo y un buffer de trabajo a través de todos los frames completos, y lo cierra una vez. Si esa forma privada no está disponible, la fábrica selecciona una implementación pública `zstdDecompressSync` con el mismo contrato de iterador y error de checksum. Una vista de trabajo emitida se consume antes de que el iterador avance.

Tras aproximadamente 500 ms de trabajo de frames acumulado, el lector asíncrono emite en el siguiente límite de frame y observa la cancelación antes de continuar. Un único frame sigue siendo una operación síncrona indivisible. Los frames completos exigen validación de fin-de-frame y de checksum; solo un frame final estructuralmente incompleto usa el decodificador de prefijos existente para la recuperación.

### Escaneo JSONL incremental

`SessionLogScanner` busca en los buffers crudos con `Buffer.indexOf(0x0A)` y convierte solo los registros completos a UTF-8 para `JSON.parse`. Transporta un registro incompleto a través de las escrituras del decodificador y copia solo ese fragmento porque el decodificador privado puede reutilizar su buffer de salida. No construye un buffer o cadena de texto plano completo, ni un array de líneas, ni un segundo array de registros analizados.

El escáner deja de retener eventos en la primera fila no analizable o hueco de secuencia, pero sigue inspeccionando los registros completos posteriores. Un `turn/end` posterior demuestra que el problema está en la región confirmada y rechaza el log. El lector Zstandard también rechaza cualquier problema de análisis, secuencia o registro parcial sin resolver tras todos los frames completos; solo un frame final estructuralmente cortado puede aportar un sufijo recuperable. Los registros completos emitidos desde ese frame cortado pasan por el mismo escáner y conservan el offset de reparación existente y la semántica de eventos recuperados.

### Admisión de la restauración

La persistencia transfiere valores JSON recién materializados a `Session.fromRestore`. Esos valores son árboles desprendidos y acíclicos, y las filas de chunks empaquetados se expanden en eventos recién asignados, así que el camino solo-restauración valida el envelope de eventos fijo con un único `for...in` y `switch`, despacha las comprobaciones de forma actual por discriminante de evento, y congela de forma iterativa el grafo poseído con un array `pending` explícito y sin conjunto de seguimiento de ciclos. La validación de superficie registra un plan de transición y confirma ese plan cuando el candidato exacto entra en el log, en lugar de planificar el mismo evento dos veces.

Las semillas prestadas que usan los caminos ordinarios de creación y fork siguen tomando una instantánea JSON y usan la congelación profunda genérica segura ante ciclos. La especialización por tanto solo cambia la restauración duradera; no debilita la aceptación de valores propiedad de quien llama.

## Alternativas consideradas

- **Una operación nativa asíncrona por frame** — rechazada porque la sobrecarga de despacho y callbacks domina los logs que contienen muchos lotes duraderos pequeños. La decodificación síncrona cooperativa paga esa sobrecarga solo en los límites periódicos de emisión.
- **Procesar el log completo de forma síncrona sin emitir** — rechazada porque impide la cancelación y el progreso del event loop durante toda la restauración. Las emisiones en límites de frame conservan un punto de observación acotado sin dividir las operaciones del códec.
- **Concatenar todo el texto plano antes de escanear** — rechazada porque retiene a la vez la entrada comprimida, el texto plano completo, la cadena UTF-8 de todo el log, los metadatos de líneas y las filas analizadas, y reescanea un prefijo de frame cortado.
- **Implementar un analizador JSON de streaming** — rechazada porque JSONL ya aporta límites de registro; la búsqueda nativa de saltos de línea más `JSON.parse` elimina los intermedios grandes sin tener otro analizador ni cambiar la semántica de JSON.
- **Usar un `WeakSet` compartido al congelar los eventos restaurados** — rechazada porque la materialización JSON no puede producir ciclos, y el conjunto añade una búsqueda por objeto reteniendo el grafo completo durante el recorrido.
- **Omitir la validación o la congelación de los valores restaurados** — rechazada porque el almacenamiento duradero es una frontera de runtime y `Session.events` promete un historial aceptado inmutable. El camino optimizado especializa esas operaciones en torno a hechos de propiedad más fuertes en lugar de eliminarlas.

## Consecuencias

En el profile representativo, el escaneo incremental redujo el tiempo de escaneo JSONL de unos 598 ms a 397 ms y el RSS máximo de unos 1,494 MiB a 1,060 MiB. La admisión de restauración redujo `Session.fromRestore` de 604–608 ms a unos 263 ms, incluida una reducción de `assertSessionEventEnvelope` de unos 77 ms a 13 ms. Estas mediciones caracterizan la entrada de la optimización en lugar de establecer límites de runtime.

El decodificador rápido depende de internals de Node sondeados en runtime, pero la incompatibilidad selecciona la implementación pública en lugar de cambiar la corrección. La cancelación se observa en torno a las emisiones cooperativas de límite de frame; el plazo no es un límite duro de tiempo de pared dentro de un frame. El array de eventos completo sigue residente porque es el log autoritativo de la sesión activa; el pipeline elimina representaciones duplicadas en lugar de paginar ese estado.

Los tests fuerzan ambas implementaciones de decodificador, comparan su orden de frames y su comportamiento ante corrupción, ejercitan la cancelación cooperativa y la recuperación de colas cortadas, y conservan los contratos existentes de envelope de sesión, superficie e inmutabilidad.
