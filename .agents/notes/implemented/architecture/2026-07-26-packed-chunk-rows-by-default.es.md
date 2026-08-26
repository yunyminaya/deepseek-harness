# Agent Note: Convertir las filas de chunks empaquetados en el layout JSONL por defecto

Status: implemented

[English](2026-07-26-packed-chunk-rows-by-default.md) | Español

## Problema

Los streams de los providers producen muchos eventos delta `assistant/chunk` de tamaño de token cuyos sobres JSON repetidos pueden pesar más que sus cargas. El log de sesión debe conservar cada chunk como un evento lógico distinto: la entrega en vivo de `session/event`, los números de secuencia, los `sourceEventSeqs`, el replay, la evidencia de cancelación y el streaming de la UI dependen todos de esos límites.

El seam de almacenamiento JSONL puede reducir ese coste de sobre sin cambiar el log lógico. Una racha de al menos tres eventos delta consecutivos del mismo bloque cabe en una fila de almacenamiento `text-chunks`, `reasoning-chunks` o `tool-call-chunks`, y la decodificación reconstruye cada evento, timestamp y número de secuencia originales. Un valor por defecto creíble debe cubrir a la vez los writers de runtime, la configuración a nivel de app, los productores de instantáneas y los fixtures confirmados; de lo contrario los tests evitan el layout que los despliegues escriben.

## Decisión

`dsh-session-persistence-jsonl` resuelve un `packChunks` omitido como `true`. El wrapper del demo ACP expone el mismo valor por defecto, y toda composición que omita el campo hereda escrituras empaquetadas. `packChunks: false` sigue siendo un modo de diagnóstico explícito del lado de escritura que almacena un evento por línea.

La lectura es incondicional y ciega al layout. Los archivos empaquetados, sin empaquetar y mixtos se cargan en el mismo `SessionEvent[]` contiguo, así que el valor por defecto no exige un cambio de versión del formato de sesión ni una migración en disco en tiempo de ejecución. La opción controla solo los lotes recién añadidos; nunca selecciona un modo de lectura.

### Eventos lógicos y filas físicas

El empaquetado permanece en el seam de almacenamiento de `dsh-session` a través de `packChunkRuns()` y `decodeStorageRecord()`. El codificador reconoce las formas exactas de los eventos delta, conserva los eventos no reconocidos verbatim y empaqueta solo rachas de al menos tres. Una fila empaquetada es vocabulario de almacenamiento, no un miembro de `SessionEventMap`: nunca entra en `Session.events` ni dispara `session/event`.

El backend JSONL empaqueta cada lote de anexión duradera. El framing sin compresión (`compression: 'none'`) y el Zstandard por defecto llevan los mismos registros de almacenamiento lógicos; elegir el modo sin compresión para fixtures revisables no desactiva el empaquetado. Los lectores de replay del repositorio y los normalizadores decodifican el formato de fila compartido en lugar de mantener codecs específicos de instantánea.

### Fixtures canónicos de instantánea

Todo fixture JSONL confirmado del formato de sesión usa la representación canónica empaquetada. `scripts/session-fixture-layout.snapshot.ts` descubre los archivos `*.jsonl` rastreados y las adiciones sin rastrear y sin ignorar de todo el repositorio, selecciona los que tienen un registro de cabecera `session` como primer registro, decodifica todos los registros del cuerpo y rechaza el contenido que difiera de la salida de `packChunkRuns()`. El inventario incluye por tanto ACP, headless, TUI, `apps/web`, sesiones padre, sesiones hijas y futuros nombres de fixture sin una lista de rutas mantenida.

Las ejecuciones de instantánea de ACP y headless cosechan la salida del backend JSONL por defecto. Los writers en modo grabación de TUI y web aplican `packChunkRuns()` a sus eventos en memoria antes de escribir los fixtures. El escenario ACP `packed-chunks` redactado corre bajo la configuración ordinaria y conserva los tres kinds de fila empaquetada; su contrato decodifica tanto su fixture fuente independiente como el fixture objetivo antes de afirmar igualdad evento por evento.

Los tests de paquete focalizados mantienen entradas sin empaquetar y con layout mixto para la compatibilidad de lectura. No excluyen el corpus de instantáneas por defecto del layout canónico.

### Convergencia de ramas en vuelo

El comando temporal [`scripts/migrate-packed-session-fixtures.ts`](../../../../scripts/migrate-packed-session-fixtures.ts) permite que las ramas en vuelo converjan tras fusionar el `master` actual: `pnpm run migrate:packed-session-fixtures` descubre el mismo conjunto de fixtures de todo el repositorio que la compuerta permanente, conserva cada línea de cabecera, decodifica los registros mixtos existentes, escribe el cuerpo canónico empaquetado, demuestra igualdad tras decodificar y demuestra idempotencia. Nunca llama a un modelo ni regenera salidas de transcript y presentación.

El comando sigue enlazado desde la política de testing y el README de instantáneas ACP mientras ramas antiguas puedan llevar ediciones de fixtures. La [propuesta de eliminación](../../proposed/process/2026-07-26-remove-packed-session-fixture-migrator.es.md) borra el CLI, el comando del paquete, esta sección transitoria y los enlaces de documentación, y luego sustituye el texto de remediación específico del comando de la compuerta permanente cuando un inventario vivo de PR abiertos muestre que toda rama afectada está fusionada, cerrada o canónica. El canonicalizador compartido y la compuerta de instantáneas siguen siendo permanentes.

### Contrato de verificación

Los tests de persistencia JSONL demuestran que omitir el campo escribe una fila empaquetada, que `false` explícito escribe un evento por línea y que ambas formas cargan eventos idénticos. Los tests del canonicalizador cubren la preservación de la cabecera, la conversión de lo no empaquetado, JSONL que no es de sesión, la idempotencia de lo ya empaquetado y la entrada malformada. La compuerta de instantáneas sin clave cubre todo fixture confirmado y todo camino de replay ensamblado; las compuertas de documentación mantienen alineados los valores por defecto de configuración y los contratos bilingües.

## Alternativas consideradas

**Invertir solo el valor por defecto del schema del backend.** Deja inconsistentes los valores por defecto de los wrappers, los serializadores directos de TUI/web, los fixtures existentes y la política futura de fixtures. Un valor por defecto solo es significativo cuando las composiciones que se envían y los tests que las representan lo comparten.

**Mantener las instantáneas sin empaquetar por legibilidad.** Las filas empaquetadas conservan cada fragmento y timestamp explícitamente, mientras el decoder y el normalizador compartidos proporcionan la inspección lógica. Mantener el mayor consumidor confirmado en un layout distinto haría que la cobertura de instantáneas evitara el camino de escritura que se envía.

**Quitar `packChunks` y empaquetar siempre.** Un único writer es más simple, pero la salida de un evento por línea sigue siendo útil para diagnósticos y para los tests focalizados de compatibilidad de layout mixto. La exclusión explícita conserva esos consumidores actuales sin debilitar el valor por defecto.

**Agrupar chunks como eventos lógicos de sesión.** Reduce el número de eventos, pero retrasa o reforma la entrega en vivo, renumera los seqs de chunk citados por los mensajes de assistant y exige que toda UI y consumidor de replay entienda otra unidad de streaming. El empaquetado físico obtiene el beneficio de almacenamiento detrás de la interfaz de persistencia existente.

**Conservar el migrador de ramas permanentemente.** El canonicalizador de solo lectura y la compuerta de instantáneas son los dueños del cumplimiento continuo. Un comando de mutación solo tiene valor mientras las ramas en vuelo sigan llevando el layout de fixture anterior, así que su vida útil queda explícitamente acotada por la propuesta de eliminación.

## Consecuencias

Las escrituras JSONL ordinarias y los fixtures confirmados usan menos filas físicas conservando el flujo exacto de eventos lógicos. Los lectores de runtime aceptan todo layout existente, y los operadores conservan un modo de diagnóstico deliberadamente sin empaquetar. Los archivos sin compresión son menos cómodos para el procesamiento de líneas por token, y las herramientas externas que tratan erróneamente toda fila posterior a la cabecera como un `SessionEvent` se topan más a menudo con etiquetas de almacenamiento; los lectores soportados llaman a `decodeStorageRecord()`.

El repositorio lleva un diff mecánico grande de fixtures, revisado mediante igualdad decodificada y la compuerta de layout canónico en lugar de la inspección línea a línea token por token. También lleva temporalmente un comando de migración de rama y sus enlaces; la propuesta de eliminación separada evita que esa ayuda de transición se convierta en superficie de proceso permanente.
