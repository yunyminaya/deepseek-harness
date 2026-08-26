# Agent Note: Registros de sesión JSONL con Zstandard

Status: implemented

[English](2026-07-19-zstandard-jsonl-session-logs.md) | Español

## Problema

El backend de persistencia JSONL conserva cada `SessionEvent` tal cual, incluidos los registros de alto volumen `assistant/chunk`. El texto crudo hace que los registros sean inspeccionables, pero gasta almacenamiento y E/S en claves JSON repetidas y en texto de modelo. La compresión debe conservar la frontera de confirmación append/fsync existente, la primera materialización segura frente a colisiones, la reparación tras un fallo y el listado solo de metadatos; reescribir un archivo comprimido entero tras cada turno descartaría esas propiedades.

La codificación también debe permanecer explícita en la frontera de despliegue. Los fixtures (datos de prueba) de instantánea y los lectores externos de líneas requieren JSONL crudo, mientras que un backend no puede adivinar de forma segura entre artefactos comprimidos y crudos en una misma raíz ni migrar en silencio datos de sesión de prelanzamiento.

## Decisión

### Configuración y propiedad del sufijo

`dsh-session-persistence-jsonl` acepta `compression?: 'zstd' | 'none'` y resuelve explícitamente la omisión a `'zstd'`. Los artefactos Zstandard terminan en `.jsonl.zstd`; `'none'` conserva la representación original `.jsonl` en UTF-8 delimitada por líneas. `SessionLocation.kind` permanece `'jsonl'`, porque ambas codificaciones llevan el mismo formato lógico de registro, y `SESSION_FORMAT_VERSION` permanece `0` bajo la política de prelanzamiento del repositorio de rechazar sin migración.

Cada raíz de persistencia pertenece a una sola codificación. Un preflight (verificación previa) de descubrimiento único rechaza cualquier sufijo contrario, y las rutas de carga dirigida, adopción en vivo, listado y materialización repiten la comprobación de sufijo pertinente después de un preflight inicialmente vacío. El error nombra el artefacto incompatible y dirige el despliegue a la configuración correspondiente o a una raíz separada. No hay migración, doble lectura, doble escritura ni fallback basado en extensión.

### Tramado de frames y ruta de escritura

El artefacto comprimido es una concatenación estándar de [frames Zstandard](https://datatracker.ietf.org/doc/html/rfc8878) independientes: un frame con checksum que contiene exactamente la línea de cabecera, seguido de un frame con checksum por cada lote de append durable. Los lotes normales del loop son confirmaciones de turno, de modo que los límites de frame conservan el punto de control de persistencia existente sin hacer que la capa de almacenamiento dependa de los tipos de evento de turno.

La compresión usa los [`zstdCompress` y `zstdDecompress`](https://nodejs.org/download/release/v22.19.0/docs/api/zlib.html) integrados de Node, disponibles en el suelo Node 22.19 del repositorio. El backend activa `ZSTD_c_checksumFlag` y por lo demás acepta los valores por defecto de Node, sin exponer ni una perilla de nivel de compresión ni una dependencia nueva. La API está marcada como experimental por Node, de modo que la puerta de compatibilidad Node 22.19, 24 y 26 ejercita exactamente ese helper.

La primera materialización comprime los dos frames iniciales antes de abrir el archivo temporal, y después escribe y hace `fsync` de ese archivo. POSIX lo publica mediante un hard link (enlace físico) seguro frente a colisiones y un `fsync` de directorio; Windows lo publica sin reemplazo mediante `MoveFileExW(..., MOVEFILE_WRITE_THROUGH)`. Los lotes posteriores se comprimen antes de abrir el destino y se añaden en EOF. Un fallo capturado de escritura o de sincronización de archivo cierra el handle de append, reabre el registro en lectura/escritura, trunca hasta la longitud de bytes anterior, sincroniza la reversión y relanza para que el coordinador pueda reintentar el lote sin cambios en ambas plataformas.

### Lectura, listado y recuperación tras fallo

Un escáner de límites de frame lee el número mágico estándar, los campos de cabecera variables, las cabeceras de bloque y los tamaños de payload, y el trailer opcional de checksum. No interpreta los bloques comprimidos. Los frames completos se validan de forma independiente por checksum y pasan por la [pipeline de restauración de sesiones grandes](2026-08-05-large-session-jsonl-restore-pipeline.es.md), que es dueña de la reutilización de decodificador, la cesión cooperativa y el escaneo JSONL incremental. Un fallo de checksum/descompresión en cualquier frame completo, una cola JSONL malformada de un frame completo o una estructura de frame inválida es corrupción y se rechaza.

El listado lee en trozos acotados solo hasta que el primer frame completo esté disponible, valida y descomprime ese frame de cabecera, y nunca lee un frame de evento. El frame de cabecera dedicado conserva por tanto el listado solo de metadatos incluso para registros de sesión muy grandes.

Un EOF dentro del frame final es una cola rasgada (torn tail) recuperable. Tras establecer ese límite con el escáner, un decodificador de prefijo dedicado usa `finishFlush: ZSTD_e_flush` para que Node emita el texto plano disponible sin exigir que el frame o el checksum se completen; conserva todo evento completo terminado en nueva línea que emita. La reparación trunca desde el byte inicial de ese frame y añade un frame nuevo con checksum que contiene los eventos completos recuperados seguidos de los cierres sintéticos de herramienta, paso y turno del coordinador. Si el desgarro ocurre antes de que ningún evento completo sea decodificable, la reparación descarta el frame parcial y conserva todos los frames completos anteriores.

### Consumidores y verificación

Los bundles de app CLI, ACP (Agent Client Protocol) y stdio exponen una configuración pass-through simétrica de `persistenceCompression`. El ensamblaje del host web y las composiciones de app ordinarias omiten la opción y usan el valor por defecto comprimido. Las composiciones de grabación y replay de instantáneas seleccionan `'none'` explícitamente porque los fixtures confirmados son entradas JSONL crudas para el replay y la normalización.

Los contratos compartidos de persistencia y coordinador se ejecutan contra ambas codificaciones. Los tests del backend cubren el tramado estándar y la interoperabilidad de checksum, el listado solo de cabecera, la reversión de append, el rechazo de codificación no coincidente, la corrupción de frames completos y los desgarros del frame final a través de cabeceras, bloques y trailers de checksum. Las smokes (pruebas de humo) de runtime por defecto, binarios compilados, headless, ACP y Python verifican el sufijo comprimido y el número mágico Zstandard o decodifican la cabecera; los tests de contenido crudo optan por excluirse explícitamente.

## Alternativas consideradas

- **Un frame por registro JSONL** — rechazado porque multiplica las cabeceras y los checksums de frame para los eventos de chunk de alto volumen y crea una frontera física ajena al lote de append durable.
- **Reescribir un stream comprimido entero tras cada append** — rechazado porque el coste crece con el tamaño del registro y el reemplazo renunciaría a la reversión append/fsync y a la mecánica establecida de materialización segura frente a colisiones.
- **Usar un compresor en streaming entre appends** — rechazado porque un estado de codificador interrumpido no deja unidades de append con checksum independiente, lo que complica el listado acotado y la reparación desde el inicio del frame.
- **Añadir una dependencia nativa externa de Zstandard** — rechazado porque el suelo de Node soportado ya proporciona el codec requerido; otro artefacto nativo ampliaría el riesgo de instalación y de empaquetado ejecutable sin añadir comportamiento requerido.
- **Exponer el nivel de compresión o conservar JSONL crudo como valor por defecto** — rechazado porque no hay evidencia de despliegue para una segunda política de ajuste, mientras que `'none'` conserva la ruta legible por líneas para los fixtures y las integraciones que la necesitan.

## Consecuencias

- Las raíces de sesión ordinarias almacenan `.jsonl.zstd` y conservan la semántica de solo añadido, fsync, reversión y recuperación de turno interrumpido.
- El JSONL crudo sigue siendo una configuración deliberada, pero cambiar la codificación exige una raíz nueva/separada o seleccionar el modo que coincida con los artefactos existentes.
- Un frame por lote durable añade sobrecarga acotada de tramado/checksum y permite el listado solo de cabecera además de la reparación desde un límite de append exacto.
- Las herramientas externas deben entender los frames Zstandard concatenados o consumir artefactos en modo crudo; la descompresión genérica de un solo disparo de Node lee solo el primer frame independiente, por lo que las lecturas del backend recorren los frames a través de la [pipeline de restauración](2026-08-05-large-session-jsonl-restore-pipeline.es.md).
- La implementación depende de la API Zstandard integrada experimental de Node sin dependencia npm; la puerta de compatibilidad de versiones soportadas hace visible la deriva.
