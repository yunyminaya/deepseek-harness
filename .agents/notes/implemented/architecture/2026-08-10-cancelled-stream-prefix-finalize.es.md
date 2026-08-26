# Agent Note: Los flujos cancelados finalizan su prefijo entregado

Status: implemented

[English](2026-08-10-cancelled-stream-prefix-finalize.md) | Español

## Problema

Un flujo cancelado puede dejar eventos `assistant/chunk` que los clientes siguen renderizando mientras `deriveMessages()` los excluye porque ningún `assistant/message` registra el prefijo entregado. Una continuación como «amplía tu segundo punto» carece entonces de texto que el usuario leyó, y una bifurcación en el turno cancelado hereda el mismo hueco.

El historial del modelo debe contener el contenido de asistente que permanece visible para el usuario tras la cancelación.

## Decisión

`ReactLoopAgent.step()` captura la cancelación mientras consume un flujo de modelo, cuando su `BlockAssembler`, las secuencias de chunks registradas y la ruta del provider identifican el prefijo entregado. Añade ese prefijo como `assistant/message` del paso con `interrupted: true`, `surfaceOp: 'append'` y `sourceEventSeqs` conteniendo exactamente los chunks registrados. El append precede a `step/end` y al `turn/end` abortado.

`BlockAssembler.interruptedBlocks()` devuelve bloques `text` y `reasoning` cerrados y abiertos con contenido no de espacios en blanco, en orden de flujo. Omite las llamadas de herramienta porque la interrupción precede al despacho y no existe resultado real; también omite los bloques vacíos y los tipos de bloque desconocidos abiertos. Un resultado vacío no añade ningún mensaje de asistente. Las finalizaciones `error` y `aborted` del provider abandonan el ámbito de consumo del flujo antes de `agent/request-error`, de modo que los fallos de provider y la cancelación durante la recuperación no confirman contenido alguno de la petición fallida.

Las definiciones de conversación de Chat y Trajectory leen `interrupted` del mensaje durable. Chat renderiza el marcador Stopped, mientras que Trajectory mantiene la petición del provider en el ciclo de vida de error tras `step/end` y conserva la secuencia durable del resultado y la procedencia. La cancelación durante la ejecución de una herramienta sigue el contrato del scheduler de herramientas porque el mensaje de asistente ya se ha confirmado: las llamadas iniciadas producen resultados reales y las no despachadas reciben resultados `ABORTED_BEFORE_DISPATCH`.

## Alternativas consideradas

**Descartar siempre el prefijo.** Evita un nuevo marcador durable pero hace que cada cancelar-y-continuar y cada bifurcación omitan contenido de asistente que permanece visible para el usuario.

**Ensamblar el prefijo desde los chunks durante la proyección.** `deriveMessages()` y las definiciones de conversación de cliente necesitarían cada una reglas de ensamblaje de interrupción, y el log no tendría ningún mensaje de asistente autoritativo para el prefijo. Esto además expande el historial del modelo más allá de los tres eventos `SurfaceEventType`.

**Conservar llamadas de herramienta completas con resultados abortados sintéticos.** Estas llamadas nunca se despacharon, así que los resultados sintéticos afirmarían un resultado de ejecución que no ocurrió y añadirían contenido que el usuario no recibió como resultado de herramienta.

**Añadir un mensaje de interrupción visible para el modelo como `[interrupted by user]`.** Puede decirle al modelo que el prefijo está incompleto, pero exige un tipo de fuente separado, una regla de proyección, un tratamiento de UI y una redacción localizada. El `turn/end` abortado durable conserva el hecho necesario para esa decisión posterior.

## Consecuencias

Las continuaciones y bifurcaciones posteriores a la cancelación incluyen el prefijo entregado. El puente ACP drena la salida de asistente ordenada antes de asentar el prompt, de modo que la actualización final de `agent_message_chunk` precede a la razón de parada cancelada.

Los errores terminales de provider siguen descartando su prefijo transmitido. Esa asimetría permanece porque un turno de error termina sin la decisión de cancelación del usuario y exige su propia política de retención.

## Pruebas

`packages/core/agent-loop/tests/cancel.spec.ts` cubre el contenido, las secuencias citadas, el orden de eventos, la paridad de la siguiente petición, la salida solo de razonamiento, la omisión de llamadas de herramienta, la cancelación durante la recuperación y el caso de prefijo vacío. `packages/llm/llm/tests/assembler.spec.ts` cubre `interruptedBlocks()`. `packages/client/ui-conversation/tests/conversation-node-definitions.client.spec.ts` y `packages/client/ui-trajectory/tests/conversation-definitions.client.spec.ts` cubren ambas proyecciones de cliente. La instantánea ACP `cancel` sin clave y la instantánea de objetivo `goal-round-driver` cubren las aplicaciones ensambladas.
