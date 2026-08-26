# Agent Note: Vincular la identidad de sesión JSONL antes de la mutación

Status: implemented

[English](2026-07-20-jsonl-storage-identity.md) | Español

## Problema

La búsqueda JSONL selecciona un log físico a partir del id de sesión solicitado a través de los directorios del proyecto, mientras que el `SessionHeader` parseado aporta los metadatos que usan las operaciones posteriores de reparación y anexo. Sin vincular esos dos hechos, un log seleccionado para la sesión A puede declarar el id o el cwd de la sesión B y redirigir una reparación o un anexo posterior a la ruta de B. El escaneo del proyecto también necesita un resultado definido cuando el mismo id codificado existe en más de un directorio del proyecto. SQLite no comparte esta ambigüedad porque su consulta por clave primaria vincula metadatos y eventos al id solicitado.

## Decisión

`loadStored(id)` es la única búsqueda de prefijo almacenado del coordinador. El backend JSONL escanea cada directorio del proyecto, exige como máximo un directorio de sesión codificada coincidente con un transcript (transcripción), parsea ese archivo y luego valida que `header.id === id` y que la ruta seleccionada o bien es igual a `logPath(root, header.cwd, header.id)` o bien la canonicalización del sistema de archivos resuelve ambas grafías al mismo transcript. `list()` aplica la misma validación de ruta y rechaza ids duplicados entre directorios del proyecto.

El coordinador afirma de forma independiente el id devuelto y compara el cwd almacenado con el cwd de una sesión en vivo antes de la reparación, la publicación de estado o la persistencia de sufijo. Mantiene una copia desacoplada de los metadatos validados; el anexo y la reparación de JSONL derivan su ruta de esa copia. La interfaz `PersistenceBackend<TornMarker>` no necesita por tanto ni una búsqueda en vivo específica del ámbito ni un tipo de localizador de almacenamiento.

Una raíz JSONL existente configurada debe ser un directorio legible cuando el plugin se carga. Una raíz ausente sigue siendo válida y se crea en la primera materialización. El backend admite un escritor en vivo por sesión; otra instancia u otro proceso del backend no debe mutar esa sesión hasta que el propietario termine el dispose y todas las escrituras se detengan.

## Alternativas consideradas

**Aplanar el almacenamiento por id de sesión.** Un espacio de nombres plano hace que la publicación duplicada colisione en una sola ruta, pero la validación de ruta y el rechazo de duplicados cierran el defecto de identidad sin hacer que la comprobación dependa de un espacio de nombres global plano.

**Llevar un localizador de almacenamiento opaco a través del coordinador.** Un localizador vincularía las mutaciones JSONL directamente a una ruta seleccionada, pero JSONL puede reproducir esa ruta a partir de los metadatos que ya ha validado. Añadir otro genérico y otro argumento a SQLite, a los backends de prueba, al anexo y a la reparación haría que cada implementación llevara un concepto que solo el backend de archivos necesita.

**Coordinar múltiples escritores en vivo.** Un servicio de coordinación dedicado, un registro global de procesos o un lock entre procesos definiría una nueva topología de despliegue en lugar de reparar la validación de identidad. La topología soportada tiene un escritor en vivo; la publicación por hard link sin sobrescritura sigue arbitrando una carrera inicial de creación con el mismo id.

## Consecuencias

Los logs JSONL no coincidentes, mal ubicados y duplicados fallan antes de la reparación o de la mutación de estado del coordinador. La búsqueda sigue siendo proporcional al número de directorios del proyecto, y la propiedad de un solo escritor en vivo sigue siendo una limitación explícita. Las pruebas del coordinador y de JSONL fijan el rechazo antes de la reparación, los bytes sin cambios de ambos logs afectados, la validación de ruta durante el listado, el rechazo de ids duplicados, las colisiones de proyectos normalizados y los alias de mayúsculas, y la validación de raíz en tiempo de carga.
