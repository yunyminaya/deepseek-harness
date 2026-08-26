# Agent Note: Presión de compactación posterior a la llamada y recuperación del desbordamiento de contexto

Status: implemented

[English](2026-07-10-after-call-compaction-pressure-and-overflow-recovery.md) | Español

## Problema

`agent/pre-step` se ejecuta antes del enrutamiento final de la solicitud y antes de que existan la salida del assistant, los resultados de las herramientas, el contexto almacenado en buffer y el steering (direccionamiento). Incluso con el prompt ensamblado y el prefijo de sesión, su vista de presión es provisional porque `agent/request` todavía puede cambiar el enrutamiento o la configuración de la llamada, y los schemas de las herramientas no están congelados con esas entradas. Añadir campos no puede hacer que el estado previo a la llamada describa una llamada completada, y acoplaría el punto de extensión genérico a la compactación.

Las llamadas exitosas no son la única señal de presión. Un provider puede rechazar una solicitud por superar su ventana de contexto antes de devolver el uso, y algunas llamadas exitosas omiten el uso. El sistema necesita, por tanto, una presión posterior a la llamada reproducible más un camino estrecho de recuperación ante fallos que conserve el error del provider siempre que la compactación no pueda demostrar un progreso útil.

## Decisión

### La presión exitosa se ejecuta en el siguiente límite de pre-step

`agent/pre-step` recibe el lote exclusivo de mensajes reclamados más `{ turn, step, signal }` y devuelve la decisión final de reject/enter. No lleva ningún campo de prompt o prefijo exclusivo de la compactación.

Compact-basic envuelve `agent/pre-step` antes de cada solicitud propuesta. En un límite de continuación, la salida del assistant anterior, cada resultado de herramienta despachado o sintético, el contexto posterior a la herramienta y el steering ya son duraderos, de modo que la política de presión ve el estado completo de la llamada exitosa sin separar una llamada de herramienta del assistant de su resultado. En el límite inicial, una sesión sin header no tiene ninguna solicitud enrutada completada y no produce trabajo de presión. Compact-basic contiene los fallos operativos, avisa y delega sin rechazar el paso propuesto.

`dsh-compaction-basic` lee el modelo enrutado más reciente exacto del header de solicitud duradero solo para establecer que existe una ruta completada, y luego pide al `ctx.tokenMeter` singleton que mida el envoltorio canónico registrado y la superficie actual. No recurre a `AgentOptions.model` para la presión automática. Una sesión sin header no tiene ninguna solicitud enrutada completada que evaluar y no produce trabajo; cualquier nombre de modelo no vacío y duradero usa el mismo estimador. Los fallos operativos de medición o resumen avisan y continúan desde la superficie duradera más reciente: el historial completo antes de cualquier reemplazo, o la superficie podada si la poda ya aterrizó.

### La recuperación de solicitudes se limita al límite final del modelo

`agent/request-error` representa los fallos terminales del límite final del adaptador. Los throws de selección de adaptador, dispatch, construcción de iterador e iteración se convierten en finalizaciones terminales `error` o `aborted` antes de que el agent loop las consuma; las finalizaciones terminales emitidas por el adaptador entran en el mismo camino. El ensamblaje del prompt, el middleware de solicitudes, el registro de solicitudes, el procesamiento de resultados, las herramientas, los listeners de paso y la limpieza siguen siendo fallos ordinarios. [Fallos terminales del stream LLM](2026-07-29-terminal-llm-stream-failures.es.md) es el dueño de este límite de normalización.

El paso fallido se cierra antes de que se ejecute la recuperación. Un listener encargado repara el estado duradero, devuelve `{ kind: 'retry' }` y detiene la delegación del waterfall (cascada de eventos). El bucle cierra entonces el turno fallido y abre un turno de reintento a partir del registro duradero sin una notificación de inactividad intermedia. La política de reintentos y los contadores de intentos siguen siendo propiedad del plugin; compaction-basic limpia su contador de desbordamiento por agent cuando la cadena alcanza `agent/settled` terminal. Ambos adaptadores de DeepSeek normalizan los fallos reconocidos de límite de contexto del provider a `CONTEXT_WINDOW_EXCEEDED`. La [decisión de la acción de reintento](../simplification/2026-07-27-request-error-retry-action.es.md) es la dueña del límite de retorno.

Si la cancelación aterriza después de que las llamadas de herramienta del assistant son duraderas pero antes de que todas se despachen, el bucle registra un par sintético de `tool/call` y `tool/result` abortado para cada llamada no despachada antes de seguir el camino de aborto normal. La superficie nunca retiene, por tanto, llamadas de herramienta duraderas huérfanas solo porque la cancelación ganó la carrera.

### CompactionEngine expone intención, no contabilidad de tokens

`CompactionEngine.compactIfNeeded(agent, trigger, signal)` acepta `trigger: 'pressure' | 'context-overflow'`. La interfaz no gana métodos de estimación ni tipos de token; `ctx.tokenMeter` sigue siendo el dueño reutilizable de la contabilidad.

Para `pressure`, compaction-basic resuelve la capacidad propiedad del adaptador del objetivo provider/modelo duradero y la política de objetivo exacto, y aplica después el umbral resultante y los presupuestos de cola retenida a un único resultado de `ctx.tokenMeter.measure()`. Por debajo de la presión, retorna sin podar. Una vez que la presión califica, el `ctx.toolResultPruner` opcional reescribe los resultados actuales sobredimensionados y compaction-basic vuelve a medir con el mismo medidor; la presión segura omite la llamada al modelo, mientras que la presión restante selecciona y resume a partir de la superficie podada. El mismo medidor singleton es dueño del precio por rangos, la contabilidad de eventos fuente citados, los recuentos de tokens ensombrecidos y el rechazo de resúmenes que no reducen. Los valores por defecto comunes siguen siendo el ratio de umbral `0.8`, el ratio de historial retenido `0.16`, el provider/modelo de resumen `''`, `maxTokens: 8192`, `compactionRetries: 1` y `auto: true`; las entradas opcionales de `modelPolicies` los sobrescriben para un par exacto de provider/modelo.

Para el desbordamiento canónico, compaction-basic no requiere metadatos de capacidad y omite la presión escalar y el presupuesto normal de tokens retenidos. Poda primero, elige después el rango de cabecera maximal equilibrado por herramientas dejando la unidad indivisible más reciente, e intenta una compactación de resumen que reduce bajo la misma señal cuando existe un rango. El listener automático hace una instantánea de `session.surface.replaceGeneration` y devuelve `{ kind: 'retry' }` siempre que la poda o el resumen la incrementen. Esto sigue siendo cierto cuando la poda aterriza antes de que un trabajo de resumen posterior lance; la cancelación sigue ganando. Un backend que devuelve un resultado sin reemplazo no puede autorizar el reintento, mientras que el progreso solo por poda puede autorizar un reintento sin un `CompactionResult`.

`maxOverflowRetries` es opcional y tiene el valor por defecto `1`; `0` deshabilita la recuperación de desbordamiento sin deshabilitar la presión. `auto: false` no registra ningún listener automático. Los errores no canónicos, los intentos agotados, una señal ya abortada, un modelo enrutado ausente, la falta de un rango seguro, la falta de cambio de generación y los throws de recuperación antes de cualquier reemplazo delegan todos en el siguiente listener. Sin recuperación posterior, el bucle informa del objeto y código de error original del provider. Un throw de recuperación después de que la generación avance autoriza el reintento desde el progreso duradero; la cancelación o la disposición sigue siendo autoritativa aunque el trabajo de recuperación se complete de forma concurrente.

El resumidor por defecto resuelve la configuración explícita, después la ruta registrada más reciente y después las agent options. Como el middleware directo de `llm/stream` puede reenrutar esa llamada auxiliar, `compaction/summary.{provider, model}` registra el objetivo final mutable de `GenerateOptions` observado después del dispatch en lugar del candidato previo al waterfall.

## Pruebas

Las pruebas unitarias cubren el límite de normalización del adaptador final, la numeración y el reinicio de los reintentos de turno cerrado, la cancelación y la disposición, el orden en los límites de paso, la presión del envoltorio enrutado, la poda condicionada a presión, el alivio solo por poda, el resumen de entrada podada, la reducción equilibrada de desbordamiento, el progreso de poda duradero antes de un fallo posterior, la prueba de generación, los límites, la delegación y el enrutamiento de llamadas auxiliares. Las pruebas de bucle real cubren el desbordamiento lanzado y en banda a través de la poda o de la compactación por resumen hasta una solicitud de reintento reconstruida.

## Alternativas consideradas

- **Añadir campos exclusivos de la compactación al pre-step** — rechazada porque la sesión duradera canónica y el medidor de tokens ya son dueños de la entrada de medición; el ciclo de vida genérico no necesita llevar un segundo envoltorio.
- **Reintentar el mismo paso numerado** — rechazada porque la recuperación añade eventos duraderos después del límite fallido. Un paso nuevo conserva el anidamiento equilibrado y la reconstructibilidad.
- **Reintentar siempre que `compactIfNeeded` devuelva un resultado** — rechazada porque un backend personalizado puede informar éxito sin cambiar el estado visible para el modelo. `replaceGeneration` es la prueba autoritativa.
- **Dejar que compaction-basic analice la redacción del provider** — rechazada porque la clasificación pertenece a los adaptadores y debe cubrir tanto la entrega lanzada como la en banda.
- **Recurrir a `AgentOptions.model` cuando no existe una ruta duradera** — rechazada porque la política automática debe describir una solicitud registrada completada. La presión sin header y la recuperación delegan sin cambios.

## Consecuencias

La comprobación de presión del siguiente pre-step describe la solicitud enrutada completada precedente, incluidos los resultados de herramientas duraderos y la entrada recién reclamada. La poda opcional sin modelo elimina el volumen predecible de salida de herramientas antes de la selección del resumen y puede crear de forma independiente progreso digno de reintento. El desbordamiento canónico proporciona el respaldo cuando no existe ningún ancla de uso exitosa. La recuperación está acotada, es propiedad de la cancelación y es monótona: reintenta solo después de un cambio visible de generación de la superficie.

El coste es el trabajo de presión en el waterfall compartido del pre-step y la clasificación de desbordamiento mantenida por el adaptador. La redacción del provider y la densidad de caracteres heurística siguen siendo riesgos de mantenimiento. La compactación de superficie todavía no puede reparar un envoltorio que solo por sí mismo supera la ventana, dividir un nodo no herramienta indivisible, ni reparar una unidad de herramienta cuyo resto no podable sigue siendo sobredimensionado. El podador opcional puede reparar un par de herramienta por lo demás indivisible cuando el contenido de resultado de herramienta con texto extraíble es el grueso.

El [ciclo de vida del pre-step reclamado](2026-07-31-claimed-pre-step-inbox-lifecycle.es.md) reemplaza el antiguo disparador post-step de esta nota. La división del servicio, el medidor de tokens independiente, el contrato de rango equilibrado, el bloqueo registrado en el registro, el reemplazo de resumen y el único hook de subclase `summarize()` permanecen sin cambios.
