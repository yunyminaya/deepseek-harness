# Agent Note: Eventos durables propiedad del goal

Status: implemented

[English](2026-07-31-goal-owned-durable-events.md) | Español

## Problema

El estado del goal y el estado del inbox tienen ciclos de vida distintos. Una mutación del goal debe sobrevivir al reinicio y al fork se admita o no cualquier contexto de modelo relacionado, mientras que un mensaje del inbox puede editarse, reclamarse, rechazarse o descartarse como parte de la programación de pasos. Codificar una mutación del goal dentro de un mensaje de inbox de ronda cero convertía la colocación en cola en el punto de confirmación del dominio y exigía el replay para conciliar inserción, admisión, identidad del mensaje, metadatos de origen y contenido renderizado.

El dominio del goal necesita estado durable, pero no necesita la propiedad de la entrada de modelo pendiente. La programación de continuaciones sigue necesitando el inbox; la persistencia del goal no.

## Decisión

`@deepseek-ai/dsh-goal` es dueño de un evento durable de sesión `goal/change`. Cada evento lleva la instantánea completa del goal posterior a la mutación o un tombstone de borrado con revisión. `GoalService` anexa ese evento de forma síncrona y después emite `goal/changed`; el replay estricto y la proyección de sesión `goal` pliegan solo `goal/change` para el estado del ciclo de vida.

`GoalMessageSource` identifica solo las rondas de continuación admitidas positivas. Un `user/message` coincidente avanza `roundsStarted`; los mensajes de usuario ordinarios y los eventos de splice del inbox no cambian el estado del goal. El paquete del goal nunca inserta, reclama, elimina ni inspecciona mensajes del inbox. `@deepseek-ai/dsh-goal-round-driver` sigue siendo responsable de poner en cola y rastrear sus propios prompts de continuación a través del ciclo de vida público del inbox.

La activación sigue siendo local al proceso. El servicio asocia la secuencia de eventos anexada síncronamente con la activación solicitada mientras su caché observa el evento; los cambios reproducidos o anexados externamente quedan por defecto desarmados. El log de sesión sigue siendo la única autoridad durable.

El dominio no proyecta automáticamente cada mutación en la entrada del modelo. Las herramientas de goal devuelven el estado actual, y los prompts de continuación incluyen el objetivo y el estado de la ronda cuando se programa trabajo de verdad. Cualquier contexto de goal futuro siempre visible es un plugin de contexto aparte que es dueño de su mensaje de inbox en lugar de un efecto secundario de la persistencia.

## Alternativas consideradas

- **Mantener los mensajes de goal de ronda cero como registro durable.** Rechazado porque acopla las confirmaciones del dominio a la mutación de la cola y exige que el plegado del goal entienda la reconciliación de reclamo y admisión aunque los resultados de la cola no puedan revertir el estado del dominio.
- **Derivar el estado del goal solo de los mensajes visibles al modelo.** Rechazado porque una mutación puede ser válida y durable sin abrir un paso, y la cancelación o el rechazo por política no deben borrarla.
- **Almacenar los goals en una base de datos aparte.** Rechazado porque el log de sesión ordenado ya aporta persistencia, replay y herencia de fork sin un segundo límite de atomicidad.

## Consecuencias

El estado del goal es independiente de la colocación en el inbox y de la admisión. El replay tiene una sola vía de mutación, las proyecciones avanzan directamente sobre `goal/change`, y los mensajes de continuación solo llevan la atribución de ronda. El modelo no recibe un mensaje `<goal_state>` de solo-mutación; el estado visible al modelo aparece a través de las herramientas de goal y de los prompts de continuación programados. Los escritores directos de la sesión siguen siendo de confianza y pueden anexar cambios malformados, que el plegado estricto y el acompañante de invariantes rechazan.

Los tests centrados de goal, goal-round-driver, command, TUI y client-fixture fijan el replay durable, la contabilidad de rondas positivas, la independencia del inbox, las actualizaciones de proyección y el comportamiento de la sesión restaurada. El test de proceso sin clave inspecciona el evento `goal/change` persistido y verifica que la creación por sí sola no inicia ninguna ronda de continuación.
