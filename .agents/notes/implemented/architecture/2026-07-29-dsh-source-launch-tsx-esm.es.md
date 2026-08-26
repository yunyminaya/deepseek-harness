# Agent Note: El lanzamiento de source de dsh a través del hook ESM de tsx

Status: implemented

[English](2026-07-29-dsh-source-launch-tsx-esm.md) | Español

> Supera el [lanzamiento nativo de source en TypeScript](../../archived/architecture/2026-07-28-dsh-native-typescript-source-launch.md): Node eliminó la capacidad sobre la que se construyó esa decisión.

## Problema

La [decisión archivada de lanzamiento nativo de source](../../archived/architecture/2026-07-28-dsh-native-typescript-source-launch.md) ejecutaba `apps/cli/src/bin.ts` bajo `node --experimental-transform-types` con un loader de rutas solo de resolución, de modo que Node era dueño de la transformación de TypeScript. Node 26.0.0 eliminó `--experimental-transform-types` (el proceso rechaza la bandera con `bad option`), dejando solo el modo strip, y el modo strip rechaza sintaxis que este grafo de source requiere: las parameter properties de Cordis vendido (`constructor(private ctx: Context)`), los decoradores `@Inject` en `vendor/hmr` y los enums/namespaces de runtime por todo `vendor/` y `packages/workflow`. El rango de engines del repositorio (`^22.19.0 || >=24.0.0`) incluye Node 26, así que la cadena de lanzamiento nativa no podía arrancar allí en absoluto — y ningún job de CI ejecutaba el vector de lanzamiento real, de modo que la incompatibilidad se publicó en silencio.

La latencia de arranque también importaba: el worker de hooks `module.register()` fuera del hilo serializaba cada resolución entre hilos (~440ms de espera de `makeSyncRequest` durante el arranque de la TUI), y el default completo de tsx (`--import tsx`) paga ~0,4s en la amplificación de resolución de su hook CJS.

## Decisión

Los lanzamientos de source de la TUI, Web y headless de `dsh` ejecutan `node --import tsx/esm`: el hook solo-ESM de tsx es dueño tanto de la transformación de TypeScript como de la proyección `paths` del tsconfig. El script raíz `dsh` usa ese vector directamente desde la raíz del repositorio; la generación de artefactos es una operación separada bajo la [decisión de separación de lanzamiento de source y build](../simplification/2026-08-12-separate-source-launch-from-build.es.md). El hook CJS permanece apagado porque el grafo de source del CLI es solo-ESM; el lanzamiento de runtime medido hasta el banner de la TUI es de ~0,7s frente a ~1,1s con el default completo de tsx y ~0,75s con la cadena nativa eliminada.

`scripts/tspath-loader.ts` y `apps/cli/src/tsconfig-paths-loader.ts` se eliminan. Con ellos se fue la regla de runtime del loader de mapear un import de workspace solo para dependencias de runtime declaradas — tsx aplica el mapa `paths` incondicionalmente. La completitud de declaraciones descansa ahora solo en las puertas estáticas: `verify-cordis-config` para los plugins bare configurados, y las restricciones de workspace para los manifests. (Esa regla de runtime encontró bugs reales: `dsh-plan-mode` y `dsh-tool-jobs` importaban `@deepseek-ai/dsh-llm` declarándolo solo en devDependencies; desde entonces corregido.)

La matriz CI node-compat (Node 22.19 y 26) gana `dsh-source-launch-smoke` (`apps/cli/tests/source-launch.compat.spec.ts`): un lanzamiento sin clave con piped-stdio del vector de runtime de producción exacto que afirma el rechazo TTY con salida no cero. Cualquier cambio futuro de Node en los hooks de módulo o en el manejo de TypeScript pone esta puerta en rojo en lugar de romper el `pnpm dsh` de los desarrolladores.

## Alternativas consideradas

**Mantener la cadena nativa en Node ≤25 y bifurcar por versión.** Rechazada: dos semánticas de transformación (amaro frente a esbuild) divergen en sintaxis límite, el launcher gana sondeo de versiones, y la matriz node-compat debe cubrir ambos caminos — mucho mantenimiento para una bandera experimental que ya cambió bajo nuestros pies. amaro también rechaza los decoradores `@Inject` que usa `vendor/hmr`, así que el camino nativo no podía arrancar la config TUI default publicada de todos modos.

**Hacer el grafo de source solo-borrable para que el modo strip de Node 26 lo acepte.** Rechazada: las parameter properties y los namespaces de valor impregnan Cordis/cosmokit/loader/schemastery vendidos; reescribirlos es un churn sin límite que se reaplica en cada sync de vendor.

**Un loader en-hilo propiedad del repo (`module.registerHooks()` + transformación con esbuild o `@swc/core`).** Rechazada por ahora: los prototipos midieron ~0,45s (el camino esbuild sin probar de extremo a extremo; SWC se rompe con la fusión de decorador + namespace de `vendor/hmr` en ambos modos de decorador), pero significa ser dueño de la corrección de transformación y de un hook de resolución que tsx ya aporta. Revisar solo si el hueco de ~0,3s se convierte en un coste real; la evidencia de profiling vive en la discusión del PR.

**Ejecutar el `lib/` compilado para Node 26 y mantener el nativo para 24.** Rechazada: pierde el bucle de desarrollo sin build en la línea de Node más nueva y mezcla los planos de source y de artefactos.

## Consecuencias

- Un único vector de lanzamiento en todo el rango de engines, incluidas las líneas futuras de Node que cambien el soporte nativo de TypeScript; la puerta de smoke lo impone por línea de la matriz.
- La transformación de TypeScript vuelve a delegarse en tsx/esbuild, invirtiendo el objetivo de la nota anterior de demostrar la transformación nativa de Node; ese objetivo es inalcanzable mientras los sources vendidos usen sintaxis no borrable y Node no publique ningún modo de transformación.
- La aplicación de dependencias declaradas en runtime en los lanzamientos de source ha desaparecido; los imports de workspace no declarados salen ahora a la superficie solo a través de puertas estáticas o de fallos de resolución en modo compilado.
- El lanzamiento de runtime mejora ~0,4s frente al default completo de tsx; ACP conserva `--import tsx` porque su grafo no fue auditado por dependencia del hook CJS y su latencia de lanzamiento no está en el camino interactivo.
