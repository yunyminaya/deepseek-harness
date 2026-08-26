# Agent Note: Cargar todos los candidatos de instrucciones con deduplicación por directorio

Status: implemented

[English](2026-07-21-instruction-load-all-dedup.md) | [中文](2026-07-21-instruction-load-all-dedup.zh.md) | Español

## Problema

El [plugin de agent-instructions](2026-06-24-workspace-context.es.md) resolvía un único archivo ganador por lista de candidatos por directorio: el primer nombre existente en `instructionFileCandidates` ganaba el slot base, y el [overlay local](2026-07-21-local-instruction-overlay.es.md) añadía un ganador más. Pero `AGENTS.md` y `CLAUDE.md` coexisten rutinariamente en el mismo directorio. En la mayoría de los repositorios uno es un symlink del otro, así que portan contenido idéntico; en repositorios en plena migración son dos archivos reales distintos que han divergido. El primero-gana descartaba silenciosamente el archivo consignado no ganador, de modo que un directorio que legítimamente llevaba dos archivos de instrucciones distintos solo mostraba uno — y cuál dependía del orden de candidatos, no del contenido. La petición era leer ambos y deduplicar solo cuando son efectivamente el mismo archivo.

## Decisión

Cada candidato existente en cada lista se carga — primero la lista base, luego la lista local — en el orden configurado. Dentro de un directorio, los candidatos cuyo contenido es idéntico byte a byte tras recortar el espacio en blanco inicial y final colapsan en el candidato más temprano en ese orden, y los bytes originales del archivo conservado se renderizan. La deduplicación es por directorio y no global, y simétrica entre las listas base y local. Recortar antes de comparar tolera una diferencia de salto de línea final o de indentación entre un archivo y su casi-copia mientras se renderiza el superviviente verbatim — la comparación «extra safe» que la petición exigía.

Los symlinks ahora fluyen uniformemente por aquí. El descubrimiento de instrucciones resuelve cada candidato y aplica stat a su destino en lugar de rechazar un symlink de componente final, así que un `CLAUDE.md` que enlaza simbólicamente a su hermano `AGENTS.md` resuelve a contenido idéntico y colapsa aquí como cualquier duplicado real idéntico byte a byte. La deduplicación por contenido renderiza por tanto el espejo por symlink común una sola vez a través de la misma vía que una copia real. La [nota de seguimiento de symlinks](2026-07-21-follow-instruction-symlinks.es.md) es dueña de esa reversión y de su riesgo residual de frontera de confianza.

## Las claves de ámbito pasan a ser por candidato

Cada par `(directory, candidateName)` es ahora su propio ámbito lógico, codificado `directory\u0000candidateName` con un separador NUL que no puede aparecer en una ruta real. `candidateScopeKey` / `decodeScopeKey` son dueños de la codificación, y `probeScopeInstruction` decodifica el nombre del candidato para leer exactamente ese archivo. Esto reemplaza la clave de ámbito con centinela de nivel que introdujo la nota del overlay: un directorio ya no tiene un «ámbito base» y un «ámbito local», sino un ámbito por nombre de candidato, así que `AGENTS.md` y `CLAUDE.md` en un directorio son ámbitos independientes que se reconcilian por separado.

Como un ámbito ahora nombra un archivo fijo, el anterior «cambio de candidato dentro de un ámbito» — un ámbito `AGENTS.md` que caía a `CLAUDE.md` y registraba el nombre antiguo en `previousPath` — ya no puede ocurrir. `previousPath` se eliminó del registro de cambios, de los metadatos serializados de `context/message` y del texto renderizado; un cambio es ahora `set`, un `replace` del mismo archivo o un `remove`. Eliminar un candidato emite un `remove` para el ámbito propio de ese candidato, dejando a un hermano distinto como ámbito independiente.

La deduplicación se aplica durante la reconciliación, no solo en la composición de línea base. Cada pasada de reconciliación reconstruye por directorio un conjunto de resúmenes de contenido recortado conservados, en orden de candidatos, así que un archivo sin cambios se elimina cuando un candidato anterior converge en su contenido, y un hermano recién duplicado se descarta o elimina. La caché de versiones almacena un `trimmedDigest` junto al resumen de contenido completo para que la vía rápida pueda reevaluar la duplicación sin releer el contenido.

## Alternativas consideradas

**Mantener el primero-gana por lista de candidatos.** Rechazado: descarta silenciosamente el segundo archivo de instrucciones consignado de un directorio y hace que el superviviente dependa del orden de candidatos y no de si los archivos difieren de hecho, que es exactamente la sorpresa que la petición quería eliminar.

**Deduplicación global entre directorios.** Rechazada: una boilerplate idéntica bajo dos directorios distintos está legítimamente en ámbito para cada uno, y el archivo más profundo debe seguir aflorando para el trabajo bajo el directorio más profundo. Colapsar entre directorios ocultaría instrucciones que el modelo debería ver.

**Comparar bytes crudos sin recortar.** Rechazado: un editor que añade un salto de línea final, o una copía que reformatea la indentación, derrotaría la deduplicación de archivos iguales en sustancia. Recortar antes de comparar es la clave tolerante que la petición exigía, y el superviviente sigue renderizando sus bytes originales.

**Seguir symlinks para que un espejo se deduplique por contenido.** Rechazado en este cambio para preservar la invariante de no-seguimiento, y luego adoptado por separado: la [nota de seguimiento de symlinks](2026-07-21-follow-instruction-symlinks.es.md) revierte esa invariante, tras lo cual un espejo enlazado se resuelve y se deduplica por contenido exactamente como un duplicado real.

## Consecuencias

Un directorio con dos archivos de instrucciones reales distintos ahora muestra ambos; un directorio cuyo segundo archivo apenas espeja el primero sigue renderizándose una vez, y el ubicuo caso del symlink queda igual. La diferencia de comportamiento visible se limita a los repositorios en transición que llevan dos archivos reales distintos. La forma de la clave de ámbito cambió de centinela de nivel a clave por candidato y `previousPath` desapareció de los metadatos durables de cambio; `dsh-session` no mantiene promesa de compatibilidad para sesiones antiguas, así que ambos son cambios libres. La fila de la caché de versiones ganó un campo `trimmedDigest`, y la reconciliación ahora compara contenido recortado por directorio, así que un archivo sin cambios puede ser eliminado por la convergencia de un hermano — una transición que el [modelo de estados](2026-06-24-workspace-context.es.md) antes no podía producir.
