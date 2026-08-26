# Agent Note: Gates de calidad mecánicos sobre directrices en prosa

Status: implemented

[English](2026-06-11-quality-gates.md) | [中文](2026-06-11-quality-gates.zh.md) | Español

La simetría hook/CI de este registro está sustituida por [Fast local Git hooks](2026-07-22-fast-local-git-hooks.es.md); CI sigue siendo la vía de aplicación exhaustiva.

## Problema

Esta base de código es desarrollada principalmente por coding agents. Los agentes siguen gates aplicados con mucha más fiabilidad que las convenciones en prosa, y «mucho trabajo» no es un argumento de coste cuando los agentes hacen el trabajo. Evidencia temprana: pruebas que no pasaban el typecheck se publicaron (vitest no hace typecheck) y solo las atrapó una revisión.

## Decisión

Cada promesa de AGENTS.md mecánicamente verificable recibe un comando que termina con código distinto de cero. CI invoca el conjunto exhaustivo, mientras que los hooks de Git reservan su presupuesto de latencia para defectos locales baratos:

- TypeScript de máxima estrictez (`noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, …); examples, tests y scripts pasan typecheck en CI vía el `tsconfig.json` raíz sin emisión, mientras que el código de packages/vendor permanece detrás de su propia frontera de project-references.
- [Oxlint](2026-07-29-oxlint-linter.es.md) con reglas TypeScript conscientes de tipos más los plugins de compatibilidad @stylistic y SonarJS, aplicando el estilo de la casa y comprobaciones de lógica duplicada local al archivo; código vendorizado excluido.
- jscpd detecta clones entre archivos en el TypeScript de producción de packages y en los scripts del repositorio; las excepciones estrechas de rango de fuente documentan implementaciones deliberadamente paralelas.
- Cobertura del 100 % por archivo en `packages/*/*/src` (v8); los guards defensivos inalcanzables llevan `/* v8 ignore */` con razones declaradas en lugar de eliminación.
- knip (código/dependencias muertos), publint (corrección de paquetes), restricciones de workspace (reglas de workspace: private, cordis peer+dev, versión uniforme, ESM), y un typecheck de consumidor NodeNext para las declaraciones de paquetes construidos.
- lefthook pre-commit aplica validación Oxlint sin proyecto y [arreglos seguros con reintento acotado](2026-08-09-oxlint-only-fix-workflow.es.md), rechaza espacios en blanco staged, y verifica el manifest de vendor; pre-push corre un typecheck incremental. CI corre la matriz completa en node 22.19/24/26 más smokes de aplicación construida para las rutas de entrada Headless, TUI, ACP, JSON-RPC, workflow y code-runtime.

## Consecuencias

- Las convenciones sobreviven a la rotación de agentes; los defectos baratos de commit/push fallan localmente y las violaciones exhaustivas fallan en CI.
- Los gates mismos son código que mantener; los cambios de configuración se revisan como cualquier cambio.
- La presión de cobertura del 100 % puede producir pruebas sin aserciones — el mutation testing es el contrapeso planificado (véase [la propuesta de mutation testing](../../proposed/testing/2026-06-11-mutation-testing.es.md)).

<!-- agent-note-format: alternatives-not-recorded (pre-format Agent Note) -->
