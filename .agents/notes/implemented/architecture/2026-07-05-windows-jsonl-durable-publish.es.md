# Agent Note: Publicación JSONL duradera nativa de Windows

Status: implemented

[English](2026-07-05-windows-jsonl-durable-publish.md) | Español

## Problema

`dsh-session-persistence-jsonl` publica un registro de sesión de forma perezosa en el primer append. El protocolo POSIX escribe un archivo temporal, le hace fsync, lo enlaza al nombre final, hace fsync del directorio padre y luego elimina el enlace temporal. El fsync del directorio padre es parte del contrato de durabilidad: un cuelgue después del cambio de espacio de nombres no debe perder el nombre final confirmado mientras deja a los llamantes creyendo que el registro de sesión se materializó.

Windows tiene operaciones atómicas de espacio de nombres, pero Node no expone allí un contrato de fsync de directorio padre equivalente al de POSIX. Tratar los fallos de sincronización de directorios de Windows como éxitos debilitaría en silencio un backend duradero. La vía de Windows necesita por tanto una primitiva de publicación distinta, no un condicional dentro del helper POSIX `syncDir`.

## Decisión

El backend JSONL se bifurca dentro de `materialize()` antes de cualquier mutación de espacio de nombres. El código compartido calcula el directorio de sesión, la ruta final del registro y la cabecera codificada más el lote inicial de eventos; POSIX y Windows ejecutan entonces protocolos de publicación separados.

POSIX conserva el protocolo existente: crea el directorio raíz, el directorio de proyecto y el directorio de sesión con fsync de los directorios padres, escribe y sincroniza con fsync un archivo temporal, publica con `link()` para que un registro final existente nunca se sobrescriba, hace fsync del directorio de sesión y luego elimina el enlace duro temporal redundante.

Windows crea los directorios faltantes mediante una publicación provisional duradera: crea un directorio hermano aleatorio bajo el prefijo constante `.dsh-mkdir-`, independiente del basename objetivo, y luego lo publica en el nombre de directorio final con `MoveFileExW(..., MOVEFILE_WRITE_THROUGH)` sin `MOVEFILE_REPLACE_EXISTING` ni `MOVEFILE_COPY_ALLOWED`. La materialización de archivos escribe y sincroniza con fsync el registro temporal, y luego publica ese archivo temporal en la ruta final con la misma llamada `MoveFileExW` de escritura a través y sin reemplazo. `koffi` es el puente Win32 mínimo para esta API; su script de instalación está permitido en `pnpm-workspace.yaml` porque el paquete incluye el loader nativo y los módulos de plataforma precompilados.

## Alternativas consideradas

**Ignorar los fallos de sincronización de directorios de Windows.** Rechazada porque informa de un primer append como duradero sin forzar la entrada de espacio de nombres publicada al almacenamiento estable.

**Usar `CreateHardLinkW`.** Rechazada porque los enlaces duros dependen del sistema de archivos, no publican directorios y no exponen ninguna opción de escritura a través.

**Usar API de reemplazo o transaccionales.** `ReplaceFileW` tiene semántica de reemplazo que entra en conflicto con el rechazo de colisiones del mismo id, y NTFS transaccional no está recomendado para diseños de aplicaciones nuevos.

## Consecuencias

El backend mantiene un único contrato externo en todas las plataformas: el primer append o bien publica un registro completo en el nombre final, o bien falla sin sobrescribir un registro existente. La división por plataforma es un detalle de implementación; las API de `SessionPersistence` y el formato lógico de registros JSONL no cambian. La posterior [decisión de codificación Zstandard](2026-07-19-zstandard-jsonl-session-logs.es.md) se aplica antes de que cualquiera de las plataformas publique los bytes opacos.

Las pruebas de Windows ejercitan la vía de publicación Win32 real en Windows nativo. El comportamiento ante pérdida de corriente sigue siendo una propiedad del contrato de API, no algo que las pruebas unitarias puedan demostrar; los invariantes comprobables son que el fsync de directorio no se llama en la materialización de Windows, que las colisiones de ruta final fallan, que los componentes objetivo de longitud máxima siguen siendo materializables, que los registros temporales se sincronizan con fsync antes de la publicación y que el registro resultante se carga con normalidad.

El append y la reparación siguen usando fsync ordinarios de manejadores de archivo en ambas plataformas. Un append fallido cierra su manejador de solo-apéndice, reabre el registro en lectura/escritura, lo trunca al tamaño previo al append y sincroniza con fsync la reversión porque Windows rechaza `ftruncate` en manejadores de solo-apéndice.
