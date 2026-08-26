# Agent Note: Mantener los Agent Notes descubribles sin un índice generado

Status: implemented

[English](2026-07-19-remove-generated-agent-note-index.md) | Español

## Problema

Un índice de Agent Notes commiteado duplica hechos ya codificados por la ruta de ciclo de vida/clase de cada archivo, la fecha del nombre de archivo y el H1. Cada rama que añade, mueve o renombra un Agent Note por lo demás ajeno reescribe el mismo archivo generado, convirtiendo ese artefacto en un punto caliente de merges previsible.

La lista cronológica centralizada añade poco valor de descubrimiento más allá de navegar el árbol de ciclo de vida/clase o buscar en el repositorio, mientras que su generador, renderizador, comando y verificación de frescura siguen siendo carga de mantenimiento.

## Decisión

El árbol de archivos por ciclo de vida/clase es el inventario de Agent Notes. [README.md](../../README.es.md) sigue siendo el punto de entrada curado y el contrato, mientras que la navegación ordinaria del árbol y la búsqueda en el repositorio proveen el descubrimiento.

`scripts/agent-note-tree.ts` es dueño de los conjuntos cerrados de ciclo de vida/clase y del caminante estructural. `verify-agent-note-classification` valida ese árbol y rechaza las ubicaciones heredadas y un `INDEX.md` en la raíz; no renderiza ni verifica la frescura de una lista centralizada.

## Alternativas consideradas

**Mantener el índice generado commiteado y resolver conflictos regenerándolo.** La regeneración vuelve mecánica la resolución de conflictos, pero no impide que ramas ajenas modifiquen el mismo artefacto ni reduce el ruido de revisión que crea.

**Ofrecer un comando de índice bajo demanda sin commitear.** Evita los conflictos commiteados, pero conserva un renderizador y un comando para una ruta de descubrimiento ya servida por la navegación del árbol y la búsqueda en el repositorio.

**Restaurar un índice mantenido a mano.** Tiene la misma contención de archivo compartido y añade errores de completitud y de orden que la generación evitaba.

## Consecuencias

- Añadir, mover o renombrar un Agent Note ya no cambia un archivo generado en todo el corpus.
- La puerta de clasificación hace menos trabajo y la topología de la puerta de documentación no gana proceso ni etapa.
- Los lectores pierden una única página cronológica y usan en su lugar el árbol de ciclo de vida/clase o la búsqueda en el repositorio.
