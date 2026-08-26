# Agent Note: La presentación de la UI de pwsh coincide con la de bash

Status: implemented

[English](2026-08-05-pwsh-ui-bash-parity.md) | Español

## Problema

La [decisión de paridad de herramientas bash para pwsh](../../implemented/feature/2026-08-02-pwsh-tool-bash-parity.es.md) hizo que `dsh-tool-pwsh` fuera intercambiable a nivel de comportamiento con `dsh-tool-bash` para la ejecución, los marcadores y los trabajos en segundo plano, pero difirió explícitamente la mitad visible para el ser humano: una llamada pwsh en primer plano completada se presentaba como una tarjeta genérica con valla `console`, mientras que la llamada completada de la herramienta bash se presentaba como una tarjeta de terminal con una píldora de estado de salida analizado. La hoja de ruta dueña de esta carencia ([Windows usa pwsh por defecto](../../implemented/feature/2026-08-01-windows-pwsh-default.es.md)) nombraba «renderizado TUI/GUI de pwsh» como fase 2, pero el paquete TUI se eliminó, dejando la superficie Web como la única UI a la que afecta la carencia.

## Decisión

`presentResult` de `dsh-tool-pwsh` refleja ahora llamada por llamada al de `dsh-tool-bash`: un resultado en primer plano completado es una tarjeta `terminal` cuyo cuerpo de salida es el texto renderizado sin marcadores y cuya píldora de estado de salida es el `exitCode`/`signal` analizados; los acuses de recibo de segundo plano y los resultados `isError` siguen siendo tarjetas genéricas con valla `console`; los resultados que no son un único bloque de texto permanecen intactos (`undefined`).

El análisis se comparte, no se duplica: `parseExitStatus`/`ParsedExitStatus` se movieron del módulo de render privado de `dsh-tool-bash` al paquete de Service Definition `@deepseek-ai/dsh-shell` (exportados desde su índice), y `render.ts` de `dsh-tool-bash` los re-exporta para que sus consumidores del plano de origen conserven una única raíz de importación. Los renderizadores de ambas herramientas emiten los mismos marcadores `[exit code: N]` / `[killed by signal: X]`, así que una inversa propiedad de la Service Definition nunca puede divergir entre los gemelos — la misma forma «compartido, no duplicado» que usó la [extracción de shell-env](../../implemented/feature/2026-08-02-pwsh-tool-bash-parity.es.md) para el registro `DSH_*`.

La UI Web no necesita código por herramienta para la tarjeta en sí: el puente de tarjetas de terminal del cliente mapea cualquier vista de resultado `card: 'terminal'` (`terminal-card-model` en `dsh-client-ui-conversation`), así que el cambio del presentador de pwsh fluye por el mismo camino de renderizado que bash ya tiene. La fila de herramienta colapsada sí recibe una entrada de clasificación de cliente: `classifyTool('pwsh')` produce ahora la fila de la familia shell (variante `bash`, con su propio título `Pwsh`) en lugar de la fila genérica «Tool call» de `others`. Una línea de navegador sin clave (`apps/web/tests/pwsh-terminal.e2e.ts`) siembra una sesión cuya llamada/resultado pwsh presenta la herramienta real en la reproducción — el api-proxy recalcula las vistas a partir de los args/resultado registrados — y fija el golden de la tarjeta de terminal, incluida la píldora de salida y el punto de estado de ejecución.

## Alternativas consideradas

**Importar `parseExitStatus` desde `@deepseek-ai/dsh-tool-bash/src/render.ts`.** Rechazado: las importaciones del workspace permanecen externas en los bundles construidos, así que `tool-pwsh` ganaría una dependencia dura de runtime de `tool-bash` en cada closure de consumidor (incluidas las composiciones que montan deliberadamente el gemelo pwsh sin bash), y que una herramienta hermana dependa de su gemela para una sola función invierte la relación entre paquetes. El movimiento del seam mantiene el contrato compartido en un paquete del que ambas herramientas ya dependen.

**Un nuevo paquete de presentación dedicado (p. ej. `@deepseek-ai/dsh-shell-present`).** Rechazado: un paquete nuevo cuesta manifests, regeneración de module-graph/catalog y superficie de README para una sola función pura; `@deepseek-ai/dsh-shell` ya está en los closures de ambas herramientas y ya es dueño de los hechos `ShellRunResult` que el análisis reconstruye.

**Duplicar el análisis en el módulo de render de `tool-pwsh` (un tercer gemelo).** Rechazado: los contratos de texto copiados derivan sin una implementación compartida ([paridad de herramientas bash para pwsh](2026-08-02-pwsh-tool-bash-parity.es.md)); el análisis y la emisión de marcadores deben co-evolucionar en un solo lugar, y el análisis es exactamente el contrato del que depende la píldora de la UI.

## Consecuencias

- Una composición de Windows que usa `dsh-tool-pwsh` muestra ahora sus llamadas de shell exactamente como se ven las llamadas de bash en la UI Web: tarjeta de terminal encabezada por cwd, salida cruda, píldora de estado de salida, punto de estado de ejecución y el tratamiento rojo de fallo en salidas distintas de cero.
- `parseExitStatus` se convierte en superficie de contrato pública de `@deepseek-ai/dsh-shell`; `dsh-tool-bash/src/render.ts` sigue re-exportándolo, así que ningún consumidor de la herramienta bash cambia.
- La fase 2 de la hoja de ruta se reduce: la TUI está eliminada (EOL), y la contraparte de tarjeta de terminal llega ahora a la superficie Web. La composición por defecto de Windows (fase 1) sigue siendo la fase pendiente.
- Verificación: `dsh-shell` es dueño de los casos límite del análisis bajo la gate de cobertura por archivo; la suite del presentador de `tool-pwsh` refleja la de `tool-bash` (round-trip limpio/distinto de cero/señal/timeout, salida con apariencia de marcador, genéricos de segundo plano/error, fallback de múltiples bloques); la suite del row-model del cliente fija la fila de la familia shell `Pwsh`; la línea web `pwsh-terminal` es el escenario ensamblado sin clave.
