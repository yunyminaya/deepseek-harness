# Agent Note: Truncate interrupted final turns on load

Status: rejected — un solo turno puede contener trabajo real sustancial, incluidos muchos pasos y salidas de herramientas grandes. Preservar los turnos interrumpidos es preferible a descartar silenciosamente esa cola al cargar.

English | [中文](2026-06-20-truncate-interrupted-turns.zh.md) | Español

## Problema

El contrato de persistencia actual preserva un turno final que se escribió de forma duradera pero nunca se cerró. Al cargar, `interruptedTurnClosers()` recorre la cola, sintetiza eventos `tool/result` de error para las llamadas de herramienta sin respuesta, añade un `step/end` cuando hay un paso abierto, añade `turn/end { kind: 'interrupted' }` y pide al backend que confirme duraderamente esa reparación. El coordinador, el backend JSONL, el backend SQLite, el vocabulario de eventos de sesión, las invariantes, la documentación y las pruebas modelan todos este camino de cierre sintético.

Es mucha maquinaria para preservar el trabajo parcial del último turno tras un fallo. Además inventa eventos que nunca ocurrieron. Un resultado de herramienta sintético es útil porque valida el historial del provider, pero también significa que el registro reanudado contiene texto visible para el modelo que ninguna herramienta produjo. El diseño actual optimiza la preservación máxima de la cola antes de que exista un producto publicado o una UX de reanudación real que demuestre que la recuperación de un turno parcial importa.

## Propuesta

Al cargar, conservar solo el último turno completado. Un backend sigue tolerando y truncando un registro final roto, pero si el prefijo duradero interpretado termina después de un `turn/start` abierto, la reparación canónica es descartar todo evento posterior al `turn/end` anterior. Nada de `tool/result` sintético, ni `step/end` sintético, ni `turn/end { interrupted }`, ni motivo `interrupted` de cierre de turno.

Esto hace que la frontera de turno persistida sea simple: un `turn/end` completado es el punto de control. Todo lo que venga después del último punto de control es cola de fallo. El siguiente prompt se reanuda desde el último transcript de provider conocido como válido, no desde un turno final reconstruido parcialmente.

## Criterios de aceptación

- `TurnEndReasonMap` elimina la variante `interrupted`.
- `interruptedTurnClosers()` y sus pruebas desaparecen.
- El hook de reparación del coordinador de persistencia trunca el estado de cola roto/abierto específico del backend sin añadir closers.
- La [documentación de persistencia de sesiones](../../../../packages/session/session-persistence/README.es.md) dice que la carga devuelve el último turno completado, sin turno final parcial.
- Las pruebas de instantánea y de contrato se actualizan junto con el comportamiento que fijan.
- La versión del formato de sesión y las fixtures grabadas se renuevan; los registros almacenados no vigentes se rechazan según la política de formato de prelanzamiento, sin camino de migración.

## Qué sacrificamos

Un fallo puede perder trabajo real del turno final: texto del asistente, llamadas de herramienta y salidas de herramienta añadidas después del `turn/end` anterior. Esa es la simplificación deliberada. El producto no está publicado, la semántica de recuperación del turno final no está demostrada ante usuarios, y un punto de control limpio de turno completado es mucho más fácil de explicar, probar e implementar. Una futura funcionalidad de «recuperar el trabajo parcial tras un fallo» debería diseñarse como una vista de recuperación explícita orientada al usuario, no como eventos sintéticos insertados silenciosamente en el transcript canónico.

## Relación

Esta es una simplificación directa de la [persistencia de sesiones](../../implemented/architecture/2026-06-14-session-persistence.es.md) y de la histórica [regla universal de cerramiento de turnos](../../archived/architecture/2026-06-15-turn-enclosure-invariant.md). También elimina gran parte de la motivación de los eventos duraderos de frontera de paso, lo que hace más pequeña la propuesta de [eliminar los eventos duraderos de frontera de paso](2026-06-20-drop-durable-step-boundaries.md).

<!-- agent-note-format: alternatives-not-recorded (pre-format Agent Note) -->
