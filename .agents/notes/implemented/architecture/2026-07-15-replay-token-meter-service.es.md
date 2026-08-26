# Agent Note: Servicio de medidor de tokens por repetición

Status: implemented

[English](2026-07-15-replay-token-meter-service.md) | Español

## Problema

La presión de contexto es útil fuera de la compactación. Un backend de compactación, un guardia de desbordamiento o un futuro plugin de política de solicitudes pueden necesitar la misma respuesta: ¿cuántos tokens consume la solicitud duradera? Mantener ese pliegue dentro de `dsh-compaction-basic` duplica la lógica de repetición, hace que la medición no esté disponible sin compactación y anima a los llamadores a reutilizar contabilidad obsoleta.

El uso del provider no es una respuesta completa. Describe una llamada exitosa bajo una envoltura de solicitud exacta, mientras que la superficie actual puede crecer, encogerse o ser reemplazada después. Las sesiones también cambian de provider y modelo, los logs antiguos pueden omitir los seqs de fragmentos detrás de un mensaje de asistente, y los campos de uso separan recuentos de entrada, lectura de caché, escritura de caché, salida y razonamiento. Un servicio útil combina por tanto el ancla exacta más reciente con un recálculo heurístico conservador y expone la revisión de log consumida por cada resultado.

## Decisión

### Un servicio concreto de la familia LLM

`@deepseek-ai/dsh-token-meter` es un paquete concreto en `packages/llm/` y registra `ctx.tokenMeter`. No se divide en interfaz y backend antes de que exista una segunda implementación. `TokenMeter` mismo expone `measure(session, requestHeader?)` y `estimateMessage(message)`; los consumidores llaman directamente al servicio singleton.

El servicio no tiene configuración. La estimación usa una heurística fija de cuatro caracteres por token más sobrecarga estructural. No hay perfiles de modelo, ajustes de capacidad, ajustes de densidad, backends de tokenizador ni estrategias específicas de lenguaje. La capacidad exacta de provider/modelo es una consulta separada propiedad del adaptador, según especifica el [Agent Note de contexto de modelo enrutado y política de compactación](2026-07-20-routed-model-context-and-compaction-policy.es.md).

### Pliegues de repetición por sesión

Cada sesión es dueña de un pliegue incremental aislado. Los pliegues activos avanzan desde `session/event`; cada lectura se pone al día a través de la cola duradera, por lo que el orden de los listeners, las sesiones sembradas y la recarga del servicio no cambian la respuesta. El pliegue rastrea instantáneas canónicas de cabeceras de solicitud completas, límites de paso, añadidos y sustituciones de superficie, uso del asistente y los seqs de fragmentos citados por cada mensaje de asistente. Un evento siguiente malformado falla transaccionalmente y permanece sin leer en lugar de mutar parcialmente el estado.

`measure(session, requestHeader?)` sincroniza el pliegue una vez y devuelve presión escalar junto con precios posicionales por nodo. `totalTokens` sigue siendo presión de solicitud y respuesta; `surfaceTokens` es el total heurístico solo de superficie y es igual a la suma de `nodes[].tokens`. Una anulación de `requestHeader` cambia solo el cálculo del precio de la presión, mientras que los campos de superficie describen siempre la sesión actual. `estimateMessage(message)` aplica la heurística fija sin estado de sesión. Cada resultado es una instantánea desacoplada y profundamente inmutable que lleva una `logRevision`. Cada medición clona los nodos actuales y es por tanto O(superficie).

El uso del provider se reutiliza solo cuando la envoltura de solicitud canónica medida coincide con el ancla de la última llamada exitosa. Cualquier cambio de provider, modelo, sistema, prefijo, herramienta o configuración de llamada provoca un recálculo heurístico completo. Los cambios de superficie siguen siendo un delta con signo respecto a un ancla coincidente, incluidos los valores negativos tras una sustitución que encoge. Una solicitud exitosa posterior reemplaza al ancla anterior, incluso a través de cambios de provider o modelo.

El uso suma las categorías disjuntas de entrada, lectura de caché, escritura de caché y salida. El razonamiento no se añade una segunda vez. Cada llamada de modelo exitosa registra un `assistant/message`, incluidas las llamadas sin contenido y con tokens máximos, con sus seqs de fragmentos anteriores exactos. Una lista `sourceEventSeqs` vacía explícita significa un flujo de provider conocido como vacío; una lista heredada ausente trata de forma conservadora la salida duradera del asistente como salida del provider.

### Compact-basic consume la medición, pero no es dueño de ella

`dsh-compaction-basic` requiere `ctx.tokenMeter`; `CompactionEngine` no gana métodos ni tipos de tokens. La configuración, la transacción de región y el resumido permanecen en módulos separados; el servicio registra sus propios listeners automáticos, mientras que `summarize()` sigue siendo su único hook de subclase. El medidor singleton calcula de forma coherente el precio de la presión, la retención, el contenido ensombrecido, los eventos de fuente citados y el rechazo de resumen que no encoge.

La compactación automática usa una medición unificada para cada decisión de umbral y retención. La transacción de región mide después de añadir su bloqueo duradero `compaction/start` y de nuevo después del resumido asíncrono, y luego compara los vectores de nodos de superficie desacoplados. Una mutación de superficie intermedia impide la sustitución; `logRevision` puede avanzar por hechos no relacionados solo de log sin invalidar un tramo seleccionado sin cambios.

La política de compactación tiene valores por defecto a nivel de servicio: proporción de umbral `0.8`, proporción de cola retenida `0.16`, `summarizationProvider: ''`, `summarizationModel: ''`, `maxTokens: 8192`, `compactionRetries: 1`, `maxOverflowRetries: 1` y `auto: true`. Los campos de nivel superior se aplican a cada destino enrutado; las entradas exactas de provider/modelo en `modelPolicies` los anulan parcialmente. La presión escala las proporciones contra la capacidad resuelta desde el adaptador propietario, y `retainTokens` puede reemplazar a `retainRatio`; la retención debe permanecer por debajo del umbral resultante. El provider y el modelo de resumido deben estar ambos configurados o ambos vacíos; un par vacío resuelve el destino de la última solicitud registrada y luego el par de `AgentOptions`.

La presión automática se ejecuta en `agent/pre-step` antes de la derivación de la solicitud y mide la envoltura duradera canónica producida bajo el provider/modelo realmente seleccionado por el `agent/request` precedente. Una sesión sin cabecera no tiene ninguna solicitud enrutada completada que evaluar y no produce trabajo; cualquier destino enrutado puede usar el estimador singleton. La recuperación canónica de desbordamiento usa la misma medición para la selección forzada de rango y reintenta solo tras una sustitución de superficie demostrada.

## Pruebas

Las pruebas unitarias cubren la estimación fija, la invalidación de envoltura y la sustitución de ancla, los límites de repetición, las instantáneas inmutables, la presión enrutada, la convergencia, la prueba de generación de desbordamiento y la reversión. Un fixture real de Loader/Include verifica la ruta de carga sin configuración del medidor de tokens y de compactación-basic en orden de dependencias.

## Alternativas consideradas

- **Mantener la estimación dentro de `CompactionEngine`** — rechazada porque la medición tiene consumidores y semántica de repetición independientes de la compactación; además forzaría a cada compactador a exponer la misma API no relacionada.
- **Dividir inmediatamente una interfaz de medidor de tokens de un backend heurístico** — rechazada porque solo existe una implementación. Un servicio concreto único preserva el seam futuro sin paquetes ni configuración especulativos.
- **Poner ventanas por modelo y perfiles de densidad en el medidor** — rechazado porque la estimación por repetición no es dueña del enrutado de modelos ni de los hechos de capacidad. El adaptador propietario de la ruta expone la capacidad, mientras que compactación-basic es dueño de la política de umbral y retención específica del consumidor.
- **Mantener mediciones escalar y de superficie separadas** — rechazado porque los llamadores necesitarían dos lecturas y emparejamiento de revisiones para una sola decisión. Una lectura solo escalar podría evitar clonar nodos por debajo del umbral, pero la API dividida introduce una ventana de carrera en el lado del llamador; la instantánea unificada acepta la clonación O(superficie) a cambio de coherencia.
- **Tratar el uso del provider como portable entre envolturas** — rechazado porque modelo, herramientas, prefijos y configuración de llamada son hechos de la solicitud. El desajuste recalcula el precio de toda la solicitud actual.

## Consecuencias

- La presión de tokens tiene un propietario consciente de la repetición que la compactación y los plugins futuros pueden compartir.
- El valor por defecto convierte el medidor en una entrada de composición de configuración cero; los despliegues configuran la capacidad en cada adaptador propietario de la ruta y anulaciones opcionales de política en compactación-basic.
- El cálculo de precio heurístico fijo sigue siendo una estimación del comportamiento del provider, no un tokenizador ni un serializador de solicitudes exacto.
- Cada medición clona la superficie posicional actual y cuesta por tanto O(superficie), incluidos los chequeos de presión que terminan por debajo del umbral.
- Las mediciones fallan ruidosamente ante límites duraderos malformados. Esto convierte la repetición corrupta en un fallo de integración con nombre en lugar de una presión que se desvía en silencio.
- La presión posterior al paso lee el límite exacto registrado de routing/herramientas/prefijo; la clasificación de desbordamiento del provider sigue siendo el respaldo mantenido por el adaptador para las solicitudes rechazadas antes de un ancla de uso exitosa.
