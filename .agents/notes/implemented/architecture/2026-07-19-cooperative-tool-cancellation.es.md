# Agent Note: Cancelación cooperativa de herramientas en el límite del registro

Status: implemented

[English](2026-07-19-cooperative-tool-cancellation.md) | Español

## Problema

Cada invocación de herramienta tipada necesita una señal de cancelación propiedad del llamador. Un `ToolExecutionInput.signal` opcional permite que los llamadores directos omitan la propiedad, vuelve opcional `exec.signal` en el cuerpo de cada herramienta y fomenta respaldos del registro que no pueden representar la vida real del llamador.

El pipeline también tiene necesidades de mutabilidad distintas en cada etapa. Las implementaciones de herramientas, la pre-política, la post-política y los observadores de resultados solo toman prestado el estado de cancelación, mientras que un envoltorio around del despacho debe reemplazar temporalmente la señal para añadir un plazo (deadline) u otro ámbito léxico de cancelación. Un único tipo público mutable o bien otorga la mutación con demasiada amplitud, o bien impide esa composición.

La cancelación puede llegar antes de la política, durante la aprobación, dentro de una espera around del despacho, después de que arranque el cuerpo de una herramienta, o mientras la post-política espera. Un único resultado `ABORTED` indiferenciado no puede decir a los consumidores durables si los efectos secundarios del cuerpo fueron posibles. Hacer competir la promise de la herramienta contra la cancelación no es un respaldo seguro, porque el trabajo abandonado del mismo proceso continúa después de que el registro informe la finalización.

## Decisión

`ToolExecutionInput.signal` es un `AbortSignal` obligatorio de solo lectura. `ToolExecution.signal` y `ToolRunContext.signal` son por tanto también obligatorios y de solo lectura. Cada llamador tipado aporta la señal que posee; el registro no ofrece ninguna sobrecarga, controlador por defecto, centinela que nunca aborta, ni ruta de ejecución de conveniencia.

`ToolDefinition.execute(args, exec)` conserva su firma existente. `defineTool()` tipa contextualmente `exec.signal` como un `AbortSignal` obligatorio, de modo que toda herramienta TypeScript registrada puede observar o reenviar la cancelación sin un cast. Los llamadores directos de primera parte y los despachos anidados de Code Mode pasan explícitamente su señal de operación actual.

El registro confía en este contrato tipado del mismo proceso. No realiza validación de `AbortSignal` en tiempo de ejecución ni añade pruebas de entrada hostil para una señal omitida o malformada. La validación permanece en los límites de parser/config, JSON de modelo/herramienta, durable/archivo, worker, proceso y cable (wire); el código JavaScript sin tipos que viole la interfaz de TypeScript no tiene contrato de compatibilidad.

### La mutabilidad sigue a la etapa del pipeline

`ToolDispatchExecution` es idéntico a `ToolExecution` salvo que su `signal` obligatorio es mutable. Solo la cascada `tools/execute` recibe este tipo. La pre-política, la post-política, los observadores de resultados, los guards y las implementaciones de herramientas reciben vistas de solo lectura de un objeto de ejecución mutable privado, propiedad del registro.

Un envoltorio around del despacho puede reemplazar `exec.signal` durante la vida de su delegación, pero no puede eliminarlo tipadamente ni asignarle `undefined`. El registro captura la señal obligatoria del llamador fuera de ese objeto mutable, fusiona cada reemplazo del envoltorio con la señal del llamador inmediatamente antes de la invocación del cuerpo, elimina los listeners con alcance de despacho tras el asentamiento y restaura incondicionalmente la señal obligatoria aguas arriba (upstream).

### Los códigos de cancelación registran si hubo despacho

`dsh-tools` exporta `TOOL_ABORTED = 'ABORTED'` y `TOOL_ABORTED_BEFORE_DISPATCH = 'ABORTED_BEFORE_DISPATCH'`. El registro anota la invocación del cuerpo inmediatamente antes de llamar a `ToolDefinition.execute()`.

`ABORTED_BEFORE_DISPATCH` lleva `{ name: 'AbortError' }` y el texto de modelo `Error: tool call aborted before dispatch`. Se aplica siempre que la cancelación impide la invocación del cuerpo, incluidos la entrada preabortada, la cancelación durante la pre-política o la aprobación, una señal de envoltorio abortada, un éxito del envoltorio adelantado por la cancelación del llamador antes de la delegación, y los hermanos del agent loop omitidos tras la cancelación del turno.

`ABORTED` lleva el texto de modelo `Error: tool call aborted` y se aplica solo después de que el cuerpo se haya invocado, incluida la cancelación mientras un envoltorio around o un listener de post-política espera tras la finalización del cuerpo. Una denegación, un fallo del envoltorio, un fallo de herramienta o un fallo de post-política siguen siendo más específicos que la cancelación genérica. Un timeout propiedad de timeout-policy sigue siendo `TOOL_TIMEOUT`, y los contextos diferidos antes de que se reemplace un resultado correcto permanecen adjuntos.

### La entrada preabortada hace cortocircuito tras la materialización

El registro primero crea el token de llamada, toma una instantánea del callback opcional de contenido final de la definición visible y toma una instantánea sin pérdida de los argumentos y los congela. Un fallo de materialización de argumentos prevalece incluso cuando la señal del llamador ya está abortada. Antes del contenido final, el registro también toma una instantánea sin pérdida del resultado candidato y convierte un fallo de instantánea del resultado en un error ordinario, de modo que el callback pueda seguir haciendo cumplir su invariante de contenido. Tras una materialización de argumentos correcta, una señal preabortada se salta `tools/pre-execute`, la aprobación, `tools/execute`, `tools/post-execute` y el cuerpo de la herramienta, y luego pasa `ABORTED_BEFORE_DISPATCH` a través de ese callback solo de contenido antes de publicar exactamente un `tools/result` congelado y autoritativo.

### El trabajo iniciado sigue llegando al reposo (quiescence)

Una vez que el cuerpo de una herramienta arranca, el registro lo espera. La cancelación llega al cuerpo a través de la señal fusionada, pero nunca entra en carrera con su promise ni la abandona. Una implementación cooperativa detiene o reenvía la cancelación y se asienta cuando su trabajo propio llega al reposo; una implementación no cooperativa del mismo proceso puede dejar el registro pendiente indefinidamente. Las capas de proceso, worker, red y provider conservan la responsabilidad de sus propios mecanismos de terminación.

Esta decisión exige la cancelación solo en el límite de invocación de herramientas. Hacer obligatorias las señales en las capacidades asíncronas alcanzables desde los cuerpos de las herramientas es una migración aparte propuesta en [Cancelación requerida a través de los seams de capacidades alcanzables desde herramientas](../../proposed/architecture/2026-07-19-required-cancellation-through-tool-capability-seams.es.md).

## Verificación

[`execution-signal-types.spec.ts`](../../../../packages/core/tools/tests/execution-signal-types.spec.ts) prueba los tipos de señal exactos requeridos, las vistas de solo lectura de observador y herramienta, la vista mutable pero requerida del around-dispatch y la inferencia de `defineTool()`. [`tools.spec.ts`](../../../../packages/core/tools/tests/tools.spec.ts) cubre la materialización preabortada, el salto de fases, las carreras de política y envoltorio, la clasificación de la invocación del cuerpo, la fusión de la señal del llamador, la precedencia de errores, la retención de contexto y el drenaje en reposo. [`tool-calls.spec.ts`](../../../../packages/core/agent-loop/tests/tool-calls.spec.ts) y [`contract-regressions.spec.ts`](../../../../packages/core/agent-loop/tests/contract-regressions.spec.ts) cubren los resultados durables equilibrados para los hermanos no despachados. [`code-mode.spec.ts`](../../../../packages/core/tools/tests/code-mode.spec.ts) y las suites de integración de primera parte cubren el reenvío explícito, mientras que [`timeout-policy.spec.ts`](../../../../packages/guard/timeout-policy/tests/timeout-policy.spec.ts) preserva la propiedad del timeout.

Ninguna prueba del registro puede demostrar que un código arbitrario de terceros del mismo proceso observe la señal o se detenga en un tiempo acotado. Las pruebas de capacidad siguen demostrando la cancelación y el reposo en el límite que es dueño de cada efecto secundario.

## Alternativas consideradas

**Mantener la señal opcional y sintetizar un respaldo.** Rechazada porque un respaldo propiedad del registro no tiene vida de llamador que representar y conserva exactamente la omisión que el tipo debería impedir.

**Validar `AbortSignal` en tiempo de ejecución.** Rechazada porque este es un límite tipado del mismo proceso, no un límite de serialización. Las comprobaciones en tiempo de ejecución duplicarían el contrato estático sin hacer exigible el uso cooperativo.

**Añadir metadatos `supportsCancellation`, comprobaciones de aridad de callback o linting del uso de señales.** Rechazadas porque ninguna demuestra que el trabajo asíncrono observe o reenvíe correctamente la cancelación. La disponibilidad es un contrato de tipos; el comportamiento sigue siendo responsabilidad de la herramienta y de la capacidad.

**Exponer un único tipo de ejecución mutable a cada etapa.** Rechazada porque los observadores y las implementaciones de herramientas solo toman prestada la señal. Los tipos por etapa hacen posible el reemplazo solo donde el pipeline es dueño de esa operación.

**Prohibir a los envoltorios around reemplazar la señal.** Rechazada porque los plazos y los ámbitos operativos anidados necesitan derivación léxica. Capturar y fusionar la señal del llamador preserva la composición sin permitir el desenganche.

**Hacer competir la promise de la herramienta contra la cancelación.** Rechazada porque informa de la finalización mientras los efectos secundarios pueden seguir vivos, violando la [regla de dispose en reposo](../../../../docs/defensive-patterns.es.md#dispose-must-reach-quiescence-not-just-request-it).

## Consecuencias

- TypeScript rechaza todo `ToolExecutionInput` que omita `signal`, toda mutación de una señal de solo lectura por parte de una herramienta u observador, y todo intento de un envoltorio around de eliminar la señal.
- Los consumidores durables pueden distinguir las llamadas cuyo cuerpo pudo producir efectos secundarios (`ABORTED`) de las llamadas que nunca entraron en el cuerpo (`ABORTED_BEFORE_DISPATCH`).
- El cambio es deliberadamente rupturista según la postura de prelanzamiento del repositorio; no queda ninguna sobrecarga de compatibilidad ni respaldo en tiempo de ejecución.
- Las herramientas cooperativas se detienen con prontitud y llegan al reposo; una implementación que ignora su señal sigue siendo observable como una llamada pendiente.
- Las interfaces de capacidad aguas abajo permanecen sin cambios hasta que se acepte e implemente la propuesta Agent Note enlazada.
