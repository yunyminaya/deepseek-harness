# Agent Note: Eliminar los archivos spill de salida completa de bash

Status: rejected — la recuperación de salida completa es un comportamiento real de bash. Un futuro servicio de artefactos/blob puede generalizarla, pero eliminar los spill files antes de ese reemplazo perdería salida de comandos útil.

[English](2026-06-20-drop-bash-output-spill-files.md) | Español

## Problema

`dsh-bash-local` mantiene acotada la salida en memoria y hace spill de los streams grandes de stdout/stderr a archivos temporales privados. Eso exige un directorio privado, creación de archivos aleatoria solo del dueño, manejo de fallos de cierre, lecturas incrementales por offset de bytes, reporte de lecturas con pérdidas, render de rutas en el texto visible para el modelo y disciplina de limpieza. La herramienta luego le dice al modelo que lea una ruta de spill local cuando la salida se truncó.

Esto resuelve un problema real, pero de forma estrecha y con fugas. Una ruta de spill es un artefacto del sistema de archivos local al proceso expuesto a la salida del modelo, no un artefacto durable del harness con acceso acotado, retención o facilidades de UI. También complica las lecturas de trabajos en segundo plano porque una lectura incremental con pérdidas tiene que apuntar a uno o dos spill files.

## Propuesta

Conservar el truncado de cola, eliminar los spill files de salida completa. Un resultado de bash contiene la cola acotada más un marcador de truncado claro; no se emite ninguna ruta. Si los usuarios necesitan recuperación de salida completa, añade un servicio genérico de artefactos/blob con propiedad explícita, limpieza y render en UI, y deja que bash adjunte las salidas grandes a ese servicio.

Esta propuesta puede aterrizar independientemente de [un runtime genérico de herramientas de larga duración](../../implemented/architecture/2026-06-20-generic-long-running-tool-runtime.es.md). Si los trabajos en segundo plano se mantienen, `bash_output` debería seguir informando que la salida se descartó, pero sin anunciar una ruta de spill.

## Criterios de aceptación

- `CollectedOutput` ya no lleva rutas de spill.
- `OutputCollector` conserva solo búferes acotados y elimina la maquinaria de archivos temporales.
- `renderResult()` informa el truncado sin una ruta del sistema de archivos.
- Las pruebas cubren el truncado de cola y ya no verifican los contenidos de los archivos de salida completa.
- La guía de seguridad en [docs/defensive-patterns.md](../../../../docs/defensive-patterns.es.md) deja de tratar los archivos privados de spill como una interfaz visible para el modelo.

## Qué renunciamos

Un modelo o un usuario no puede recuperar el prefijo omitido de una salida de comando enorme desde un archivo temporal. Eso es aceptable hasta que exista un servicio de artefactos real. La ruta de spill actual es demasiada maquinaria hecha a medida para una funcionalidad cuyo ciclo de vida y permisos no están diseñados.

<!-- agent-note-format: alternatives-not-recorded (pre-format Agent Note) -->
