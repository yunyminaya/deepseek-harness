# Agent Note: Las pruebas de mutación como contrapeso de la cobertura

Status: proposed

[English](2026-06-11-mutation-testing.md) | Español

## Problema

La puerta de cobertura del 100 % por archivo ([la decisión de quality gates](../../implemented/process/2026-06-11-quality-gates.es.md)) demuestra que cada línea se *ejecuta* bajo prueba — no que alguna aserción notaría si la línea estuviera mal. Bajo pruebas escritas por agentes, la presión de cobertura puede producir ejecución sin aserción. Las pruebas de mutación miden lo que la cobertura no puede: si la suite *mata* bugs inyectados deliberadamente.

## Propuesta

Stryker (`@stryker-mutator/vitest-runner`) sobre `packages/*/src`:

- **Ejecuciones incrementales por PR** (solo archivos cambiados) como trabajo de CI — lo bastante rápidas para controlar fusiones una vez ajustadas.
- **Ejecuciones completas nocturnas** con una puntuación de mutación rastreada; empezar grabando, luego fijar el umbral en la línea base observada y apretarlo hacia arriba (la misma política que la cobertura: los umbrales solo se endurecen).
- Los mutantes supervivientes son elementos de trabajo: un agent toma un superviviente, escribe la prueba que lo mata, repite — un bucle autónomo bien formado.
- Los mutantes equivalentes (que preservan comportamiento de forma demostrable) reciben exclusiones anotadas con motivos, reflejando la política `/* v8 ignore */`.

## Plan

1. Añadir la configuración de Stryker acotada a un paquete (llm — el más pequeño y algorítmico) y medir el runtime.
2. Extender a todos los paquetes; grabar las puntuaciones de línea base en la configuración.
3. Conectar el trabajo nocturno; añadir el trabajo incremental por PR cuando el runtime sea aceptable.

## Criterios de aceptación

- Una configuración de Stryker corre sobre `packages/*/src` con el runner de vitest; un trabajo nocturno registra la puntuación de mutación y un umbral de trinquete falla la ejecución cuando la puntuación cae por debajo de la línea base grabada.
- Las ejecuciones incrementales por PR controlan las fusiones cuando el runtime es aceptable — o se mantienen explícitamente solo en el modo nocturno, con ese resultado registrado aquí.
- Los mutantes equivalentes llevan exclusiones anotadas con motivos, reflejando la política `/* v8 ignore */`.

## Riesgos

Runtime: las pruebas de mutación son caras; la cobertura del 100 % por archivo ayuda (cada mutante al menos se alcanza). Si las ejecuciones por PR siguen siendo demasiado lentas, mantenerlas solo en el modo nocturno y confiar en el trinquete de puntuación.

<!-- agent-note-format: alternatives-not-recorded (pre-format Agent Note) -->
