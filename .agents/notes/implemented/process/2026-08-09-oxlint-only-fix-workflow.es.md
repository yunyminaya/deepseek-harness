# Agent Note: Flujo de trabajo de arreglos solo con Oxlint

Status: implemented

[English](2026-08-09-oxlint-only-fix-workflow.md) | [中文](2026-08-09-oxlint-only-fix-workflow.zh.md) | Español

## Problema

La [migración del linter del repositorio](2026-07-29-oxlint-linter.es.md) conservó una invocación de ESLint solo de formateo porque el puente del plugin de JavaScript de Oxlint se trataba como solo de validación. La cadena de herramientas de Oxlint fijada ejecuta los fixers seguros proporcionados por `@stylistic/eslint-plugin`, así que el formateador separado duplica un límite de configuración, el arranque de un comando y las dependencias directas de `eslint` más `@typescript-eslint/parser`.

Una única invocación de Oxlint no es un reemplazo equivalente. Los arreglos superpuestos de plugins pueden aplicar un cambio y dejar un diagnóstico recién expuesto; el fixture de `semi` y `object-curly-spacing` del repositorio exige una segunda pasada antes de quedar limpio. El flujo de trabajo debe reintentar ese caso sin imprimir diagnósticos obsoletos de la primera pasada.

## Decisión

Todos los flujos de trabajo de lint y de arreglos del repositorio invocan Oxlint a través de [`scripts/run-oxlint.ts`](../../../../scripts/run-oxlint.ts). La validación normal sigue siendo un proceso con salida heredada. Una invocación que contenga `--fix`, `--fix-suggestions` o `--fix-dangerously` captura el primer resultado de Oxlint; el éxito emite su stdout y stderr en sus canales originales, mientras que una ejecución completada con código distinto de cero descarta sus diagnósticos potencialmente obsoletos y ejecuta una vez más el mismo comando con salida heredada. El runner vuelve a lanzar una señal del hijo en lugar de reintentar o convertirla en un código de salida, y la finalización del segundo proceso es definitiva.

El script de paquete `lint:fix` y el trabajo de lefthook en staged usan ese runner directamente. El perfil raíz consciente de tipos sigue ignorando las formas de fixtures de TypeGraph preservadas que `oxlint-tsgolint` no puede analizar; el perfil de staged sin proyecto vuelve a incluir ese directorio, lleva sus excepciones intencionales de `any` y comillas, y aplica sus arreglos de estilo antes de la pasada completa de arreglos consciente de tipos. La configuración de ESLint solo de formateo y las dependencias de desarrollo directas de `eslint` y `@typescript-eslint/parser` están ausentes. `@stylistic/eslint-plugin` y `eslint-plugin-sonarjs` siguen siendo plugins de JavaScript de Oxlint porque conservan reglas aplicadas; pnpm todavía instala ESLint como su peer declarado, pero ninguna configuración ni flujo de trabajo del repositorio lo invoca.

## Verificación

El contrato de lint ejecutable empuja una violación de estilo deliberadamente superpuesta a través del runner del repositorio y exige una salida con éxito más los bytes finales exactos. El mismo contrato fija el conjunto completo de reglas de Stylistic, la cobertura de fixtures de TypeGraph sin proyecto, los scripts de paquete, el comando del hook en staged, la configuración de formateador eliminada y la ausencia de dependencias directas de parser y runner de ESLint. Las sondas ejecutables existentes siguen cubriendo los plugins de compatibilidad Stylistic y SonarJS, la validación de staged sin proyecto y el descubrimiento de proyectos consciente de tipos.

## Alternativas consideradas

**Conservar la pasada de ESLint solo de formateo.** Esto conserva el comportamiento multipaso integrado de ESLint pero retiene un segundo runner, una config de formateo duplicada y dependencias directas después de que Oxlint pueda ejecutar los mismos arreglos de plugins.

**Ejecutar Oxlint una vez con `--fix`.** Es más simple, pero los arreglos seguros superpuestos pueden dejar el comando parcialmente formateado y con código distinto de cero aunque otra pasada idéntica lo complete.

**Adoptar Oxfmt.** Una migración de formateador cambia el contrato de salida del repositorio y crearía un diff de formateo no relacionado. Sigue siendo una decisión separada de eliminar la ruta de ejecución de ESLint redundante.

**Eliminar los plugins de compatibilidad de JavaScript.** Esto eliminaría su grafo de peers de ESLint pero también descartaría las reglas aplicadas de Stylistic y SonarJS. La pureza del árbol de dependencias no justifica debilitar el contrato de calidad.

## Consecuencias

Los colaboradores, los scripts de paquete, los hooks y CI tienen un único runner de lint y una única configuración de reglas. El perfil de staged repite la lista de ignores de la raíz para poder volver a incluir los fixtures solo de formateo sin exponer archivos vendored o generados. Una invocación de arreglos completada con código distinto de cero siempre paga un reintento, incluido un error estable e irreparable. La primera pasada almacena en búfer cada flujo de salida hasta 64 MiB para que los diagnósticos obsoletos no se impriman cuando el reintento tiene éxito; los fallos de creación y captura de procesos, incluido el rebasamiento de ese límite, aparecen de inmediato sin reintento.

El lock de dependencias todavía puede contener ESLint a través de la resolución de peers de los plugins. Eliminar ese paquete transitivo exige reemplazos nativos o una decisión de formateador que también reemplace los plugins de compatibilidad; no forma parte de la simplificación del flujo de trabajo.
