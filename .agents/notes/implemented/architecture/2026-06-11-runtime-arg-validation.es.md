# Agent Note: Runtime arg validation at the model boundary

Status: implemented

[English](2026-06-11-runtime-arg-validation.md) | Español

## Problem

`defineTool` ([el DSL unificado de esquemas](2026-07-20-unified-json-value-schema-dsl.md)) da a los autores de herramientas un `execute(args)` tipado a través de la asignación `InferArgs<S>`. Pero ese tipo es una afirmación en tiempo de compilación sobre un valor que llega en tiempo de ejecución como JSON generado por el modelo: nada forzaba al modelo a honrar el esquema, por lo que una llamada malformada — falta una clave obligatoria, una cadena donde se declaró un número, o un literal fuera del conjunto declarado — llegaba a `execute` solo con nombre tipado. El cuerpo de la herramienta entonces fallaba por la forma incorrecta o se comportaba mal silenciosamente.

## Decision

`validateArgs(spec, args): string[]` compila un `ParameterSchemaSpec` y delega al walker compartido `validateJsonSchemaValue()`, devolviendo violaciones legibles por humanos para una declaración bien formada. `defineTool` captura el esquema de parámetros compilado en el momento de la definición y ejecuta esa validación antes del cuerpo tipado; las violaciones lanzan `ToolArgsError` (`INVALID_ARGS`), que el registro devuelve como un resultado de error que el modelo puede corregir.

El validador y el compilador, por tanto, comparten exactamente la misma semántica: la raíz de parámetro implícita es un objeto abierto; las claves obligatorias provienen solo de `required: true`; los valores predeterminados siguen siendo anotaciones; los objetos anidados explícitos honran su apertura declarada; los arrays recursan a través de `items`; las restricciones de literales escalares son correctas en tipo; y `oneOf` acepta exactamente una rama coincidente. Las herramientas registradas de forma sin procesar son dueñas de su propia validación de entrada.

## Consequences

- El modelo recibe retroalimentación accionable sobre sus propias llamadas malformadas en lugar de un fallo opaco, cerrando la brecha entre la promesa de `InferArgs` y la realidad en tiempo de ejecución.
- El validador y `InferArgs` deben mantenerse de acuerdo; [una prueba de propiedades](../testing/2026-06-11-property-based-testing.md) genera argumentos que satisfacen una especificación y afirma que pasan `validateArgs` (con corrupciones dirigidas rechazadas), cerrando mecánicamente ese riesgo de deriva.
- `ToolArgsError` subclase `HarnessError` de la [taxonomía de errores estructurada](2026-06-11-structured-error-taxonomy.md), manteniendo su campo `code`; las llamadas que leen `.message` no se ven afectadas por la jerarquía.
- El costo de validación es despreciable junto a una llamada al modelo.

<!-- agent-note-format: alternatives-not-recorded (pre-format Agent Note) -->