# @deepseek-ai/dsh-subagent-in-process-driver

[English](README.md) | Español

Este paquete es el driver de ejecución compartido de los dos providers en proceso. Spawn no pasa semilla de sesión; fork pasa el prefijo de turnos completados del padre. Todo lo demás — profundidad, creación del hijo, personalización opcional del hijo, lectura del resultado, cancelación y liberación — tiene aquí una única implementación.

## Contrato de arranque

`startInProcessRun(request, options): Promise<SubagentRun>` solo se cumple después de que el hijo se publica en `ctx.agents`. Un arranque rechazado ya ha llevado a la quiescencia la transacción de creación no publicada de la fábrica de agents, así que el llamador nunca recibe un manejador a medio crear.

El driver sigue esta secuencia:

1. Valida la profundidad del padre y el `maxDepth` absoluto opcional, luego deriva la profundidad del hijo como la del padre más uno y la persiste en la cabecera de sesión del hijo.
2. Llama directamente a `parent.ctx.agents.create`, pasando la señal de solicitud requerida a la transacción de creación de la fábrica.
3. Durante la ventana de configuración no publicada de esa transacción, instala la persona solicitada, la restricción de herramientas y el runtime de salida estructurada.
4. Publica el hijo, retiene el `AgentHandle` devuelto y conduce una tarea con `child.followup(prompt)` seguido de `child.whenIdle()`.
5. Lee la salida propia del hijo — su último mensaje de asistente no vacío (se omite un mensaje de contenido vacío que registra uso), o su texto de asistente acumulado cuando no existe tal mensaje — y el motivo de turno persistente final de la ejecución completa del hijo, excluyendo cualquier semilla de fork.

El hijo recibe el linaje de directorio de trabajo/sesión del padre y hereda el provider, el modelo y el tope de tokens de salida del padre salvo que `request.agentOptions` los anule. Obtiene un ámbito de registro plano nuevo: la propiedad del padre no importa las restricciones de herramientas del padre ni establece un subconjunto de autoridad.

Este límite de resultado es válido porque el provider es dueño de un ciclo de vida de hijo aislado desde la publicación hasta la quiescencia. El direccionamiento (steering) enviado durante ese ciclo de vida pertenece a la ejecución del hijo; el provider no finge que la continuación inicial por sí sola es dueña de su salida.

El driver aplica la [política delegada](../subagent/README.es.md#delegated-policy) del seam a través de los helpers compartidos de agent hijo: captura la anulación explícita de sandbox del padre y la fijación de aprobación `'never'` antes de la creación del hijo y añade los eventos etiquetados con fuente durante la configuración no publicada, después de cualquier historial de fork y antes de la publicación de la sesión. Véase la [decisión de política de delegación](../../../.agents/notes/implemented/feature/2026-07-25-subagent-policy-inheritance.es.md).

## Cancelación y propiedad

La señal de solicitud requerida cubre tanto el arranque como la ejecución en vivo. Antes de la publicación, `AgentCreationTransaction` la observa, revierte y rechaza. La fábrica desacopla ese listener de solo creación antes de devolver; el driver comprueba la señal una vez más de inmediato antes de instalar un listener mínimo de ejecución en vivo, cerrando la carrera de traspaso. Después de la publicación, la cancelación cancela al hijo.

Tras el cumplimiento, el llamador es dueño de la ejecución. La descarga del plugin del provider no la revoca. `dispose()` elimina el listener de cancelación en vivo, registra la cancelación y delega en el `AgentHandle.dispose()` devuelto, cuya transacción de quiescencia memoizada detiene el loop, elimina el agent y la sesión y deshace los registros con ámbito. La cancelación se hace dueña de cada resultado en vuelo no completado e informa `aborted`; un turno ya completado permanece completado.

## Entradas de spawn y fork

`InProcessRunOptions` es `{ seed?: SessionEvent[] }`. Spawn lo omite. Fork suministra un prefijo equilibrado de turnos completados y registra su longitud para que el lector de resultados nunca confunda un mensaje del padre sembrado con la salida del hijo.

La aplicación de la profundidad es interna a `startInProcessRun`: lee la profundidad del padre mediante `delegationDepthOf` (el `SessionHeader.delegationDepth` persistido es autoritativo; el `AgentOptions.subagentDepth` de runtime puede aumentar pero nunca reducir, así que un hijo reanudado conserva su presupuesto), trata la ausencia como profundidad de nivel superior cero, rechaza los valores almacenados malformados e informa de un intento de profundidad de hijo por encima de `maxDepth`. Una profundidad no representable por encima del dominio de enteros seguros es un `RangeError`. La profundidad del hijo se escribe en la cabecera del hijo, así que sobrevive a la persistencia y a la reanudación.

## Salida estructurada

`attachStructuredRuntime(childCtx, schema)` instala el contrato completo en el ámbito del hijo:

- Una herramienta `structured_output` registrada con el schema solicitado valida y escenifica el valor del modelo.
- Una sección de system-prompt de orden 190 dice al hijo que la llamada de herramienta es la respuesta terminal.
- Ambas contribuciones son registros ordinarios con ámbito de hijo. Un listener experto de `system-prompt/assemble` puede reemplazarlas y por tanto es dueño de preservar el protocolo de salida estructurada para ese hijo.
- Un observador de `tools/result` confirma un valor escenificado solo después de que el resultado final autoritativo de herramienta de esa ejecución tiene éxito, incluido el resultado `run_code` envolvente para el sub-despacho de Code Mode.
- Una salvaguarda de herramienta monótona bloquea las llamadas posteriores tras la captura, y el marcador `concludeTurn()` de la ejecución de salida estructurada termina el turno después de que el resultado se confirma.

Un turno limpio que nunca confirma el valor estructurado requerido informa `error`; el driver no vuelve a hacer prompt. Todos los registros viajan en la fibra del hijo y desaparecen con ella.

## Experiencia del modelo

### Solicitud de agent hijo

#### Lo que ve el modelo

El driver compartido envía la tarea verbatim como mensaje de usuario del hijo y, cuando se solicita, ensombrece la persona y restringe los schemas de herramientas globales, la búsqueda, la ejecución y los enlaces de SDK de Code Mode en el ámbito nuevo del hijo no publicado; las restricciones del padre no se heredan, y las secciones independientes de guía de herramientas permanecen. Spawn no suministra historial; fork suministra su semilla equilibrada.

#### Efecto en tokens

La entrada del hijo está aislada del padre y crece con los propios pasos del hijo. Una persona cambia el texto de prompt repetido; el filtrado cambia el coste de schema o de SDK generado, pero no la guía registrada de forma independiente.

#### Efecto en KV Cache

Independiente de la caché de solicitudes del padre. El historial posterior del hijo es de solo añadido, mientras que los cambios de persona, filtro de herramientas, SDK generado, provider o modelo establecen un prefijo de hijo distinto.

### System prompt, schema y resultados de la salida estructurada

#### Lo que ve el modelo

Una ejecución estructurada añade la instrucción de salida estructurada de abajo. También añade una definición `structured_output` con ámbito de hijo con la descripción exacta `Report your final structured result. Call this exactly once, when your answer is complete; the arguments must match this tool's parameter schema exactly.` y el schema solicitado. Esta definición solo de runtime queda fuera del [mapa de paquetes de herramientas](../../../docs/tool-catalog.es.md#tool-package-map) generado y distribuido. Su confirmación canónica es `{ recorded: true }`, representada como `Structured output recorded.`; una llamada posterior se convierte en ``Error: structured output already recorded: the run is complete, so `<tool>` is not executed``.

##### Instrucción de salida estructurada

```markdown
When you have your final answer, you MUST report it by calling the `structured_output` tool with arguments matching its parameter schema exactly. Do not finish with a plain text answer: only the tool call counts as your result.
```

#### Efecto en tokens

Los tokens fijos de instrucción y de capacidad los paga solo ese hijo. El texto del resultado entra en el historial del hijo, mientras que solo el valor capturado se convierte en el resultado del padre.

#### Efecto en KV Cache

Estable en prefijo dentro del hijo mientras la instrucción y el schema de salida estructurada no cambien. Cambiar el schema o la capacidad puede invalidar la caché del hijo desde ese segmento temprano; los resultados se añaden en los historiales del hijo y del padre.

### Error de arranque del padre, indirectamente

#### Lo que ve el modelo

A través de `dsh-tool-subagent`, un estado de profundidad no válido se convierte exactamente en `Error: agent subagentDepth must be a non-negative safe integer`, `Error: subagent child depth exceeds the safe-integer range` o `Error: subagent depth <attempted> exceeds maxDepth <max>`. Una cancelación previa a la publicación pasa su motivo de cancelación por el envoltorio `Error: <message>` del registro.

#### Efecto en tokens

Cero tokens en un arranque con éxito; solo la llamada de herramienta fallida del padre retiene este texto.

#### Efecto en KV Cache

De solo añadido; el contenido recién visible sigue el prefijo de solicitud reutilizable y no invalida las entradas existentes de la caché KV.

### Resultado del padre, indirectamente

#### Lo que ve el modelo

El driver extrae solo la última salida de asistente propia del hijo o el valor estructurado capturado; los mensajes del padre sembrados y el trabajo intermedio del hijo no se convierten en el resultado.

#### Efecto en tokens

El padre recibe un resultado dependiente de los datos a través del Consumer; todos los demás tokens del hijo permanecen en la sesión del hijo.

#### Efecto en KV Cache

De solo añadido; el contenido recién visible sigue el prefijo de solicitud reutilizable y no invalida las entradas existentes de la caché KV.

## Limitaciones conocidas y trabajo diferido

- **Las ejecuciones no exponen `sendMessage`/`resume`** — las capacidades de runtime opcionales están ausentes en las ejecuciones en proceso.
- **La captura estructurada acepta solo el subconjunto de schema de `defineTool`** — las construcciones de JSON Schema no soportadas fallan antes de que el hijo se cree; un provider que necesite un vocabulario de schema más amplio requiere un runtime distinto.
