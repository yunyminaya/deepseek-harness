# Agent Note: Publicación de chunks de razonamiento coalescida por frame y validación de estrés en navegador

Status: implemented

[English](2026-08-03-opt-in-reasoning-chunk-browser-stress.md) | Español

## Problema

Los streams de razonamiento largos producen continuamente grandes cantidades de eventos `assistant/chunk`. Cada evento crudo debe ordenarse, registrarse y plegarse en `PartialAccumulator` para preservar la fidelidad del replay y la completitud del contenido final; React, sin embargo, solo necesita el resultado acumulado actual, no cada estado intermedio dentro de un frame del navegador.

Cada `yield` en un stream asíncrono puede crear un nuevo límite de microtarea, así que `Notifier.markDirty()` respaldado solo por batching de microtareas degenera en reconstruir una `ConversationSnapshot`, notificar a `useSyncExternalStore` y correr un render de React por cada chunk. Incluso con la fila Think en vivo colapsada, 100 000 chunks de razonamiento pueden abrumar al hilo principal con trabajo de reconciliación, commit y layout. El límite de rendimiento debe sentarse entre la ingesta de sesión y la publicación de React; no puede ocultar el problema ralentizando al productor ni descartando eventos crudos.

## Decisión

`Session.acceptLiveEvent()` añade cada evento crudo inmediatamente y actualiza sincrónicamente el transcript, `PartialAccumulator` y el resto del estado derivado de la sesión. Los chunks visibles `block-start`, `text-delta`, `reasoning-delta`, `tool-call-delta` y `block-end` se publican a través de `Notifier.markFrameDirty()`: el primer cambio programa un `requestAnimationFrame`, los chunks posteriores solo continúan actualizando el acumulador, y el callback del frame reconstruye una instantánea acumulada desde el estado más reciente y notifica a los suscriptores una vez. `usage`, `finish` y los chunks invisibles desconocidos permanecen en la ventana de eventos pero no disparan notificaciones redundantes de React. Las comprobaciones de sesión e historial comparten la misma clasificación de chunks visibles.

`Notifier` rastrea el trabajo de publicación pendiente con un tipo de programación y un marcador de generación. Los eventos estructurales ordinarios continúan publicándose en una microtarea a través de `markDirty()`; si un mensaje finalizado, un evento de herramienta o un error llega mientras una publicación de frame está pendiente, la microtarea lo supera y el callback de frame antiguo queda invalidado por su desajuste de generación. `notifyNow()` igualmente invalida la programación antigua para preservar el eco síncrono de las entradas controladas. Los entornos sin `requestAnimationFrame` caen de vuelta al batching de microtareas. Un evento de finalización puede saltarse un parcial intermedio que aún no ha aparecido, mientras que el contenido final publicado y la secuencia de eventos crudos permanecen completos.

Mantener la fila Think en vivo anclada horizontalmente al final del texto acumulado es pura alineación visual y no requiere lecturas de layout síncronas en cada commit de React. Un scheduler en el componente coalesce peticiones consecutivas en una actualización cada tres frames, lee `scrollWidth` y `clientWidth` del DOM más reciente y actualiza `scrollLeft` directamente a la posición más reciente; la cadencia visual fija mantiene legibles los cambios de resumen sin permitir que los smooth-scroll animations del navegador se acumulen. Este throttling se aplica solo al resumen horizontal de Think y no retrasa el scroll del cuerpo de Chat, el anclaje de history-prepend ni el `scrollIntoView` disparado por el usuario.

`pnpm run test:web:stress` sigue siendo la evidencia de rendimiento de navegador sin clave y opt-in. La sesión determinista `?fixture` emite 100 000 eventos `reasoning-delta` a una cadencia independiente del pintado, y un marcador de terminal prueba que los eventos cruzan la reducción de sesión de producción y llegan a la fila Think en vivo; un heartbeat de 50 milisegundos y un evento de DOM pre-programado miden los stalls del hilo principal y la latencia de interacción, respectivamente, con un presupuesto de 250 milisegundos para identificar regresiones claras. `DSH_WEB_STRESS_HEADFUL=1` permite a los desarrolladores perfilar el mismo escenario en un navegador visible con el panel Performance. El carril de estrés es evidencia para el diagnóstico manual de rendimiento y la aceptación de arreglos, no una puerta de CI por defecto ni un sustituto de los tests unitarios de programación determinista.

Los tests enfocados fijan el coalescing por frame de `Notifier`, la preempción de eventos estructurales, los callbacks invalidados y el fallback sin rAF, y prueban en la capa `Session` que un frame publica el texto acumulado más reciente solo una vez y que la finalización no va seguida de una notificación duplicada de un callback de frame obsoleto. Los tests unitarios de fixtures pequeños siguen fijando la validación de entrada, el ritmo de llegada externo, el rechazo de concurrencia, el recuento exacto de eventos y la entrega del marcador terminal sin meter la carga de 100 000 chunks en las suites de test por defecto.

## Alternativas consideradas

**Transiciones de React, valores diferidos o throttling de componentes aplicado a instantáneas.** Rechazado: la fuente de la sesión seguiría notificando a `useSyncExternalStore` por cada chunk, el render de React ya ha ocurrido antes de que un componente decida diferir la visualización, y varios componentes que consuman la misma instantánea tendrían que implementar cada uno la estrategia. El throttling de seguimiento visual para el resumen de Think ocurre después de la publicación de la instantánea y solo reduce la frecuencia del layout síncrono; no implementa la política de publicación de datos.

**Descartar, muestrear o concatenar chunks crudos en la capa de ingesta o registro.** Rechazado: los eventos crudos `assistant/chunk` son hechos de sesión reproducibles; cambiarlos reduciría la fidelidad de diagnóstico y de UI y mezclaría la política de frecuencia de visualización en la capa de datos autoritativa.

**Solo batching de microtareas.** Rechazado: las operaciones `yield` asíncronas consecutivas pueden drenar la cola de microtareas entre chunks adyacentes, haciendo que el batching de microtareas se aproxime a una notificación por chunk.

**Marcar el ritmo del productor de test por frames de animación.** Rechazado: el productor se ralentizaría cada vez que el render se ralentizara, dando a la página una backpressure implícita ausente de un stream de red real y enmascarando la inanición del hilo principal.

**Un modelo en vivo o un stream de bytes HTTP grabado.** Rechazado: los modelos en vivo son no deterministas, y una grabación HTTP/SSE no mejoraría la aserción objetivo. El fixture en memoria preserva los eventos de sesión asíncronos individuales, la reducción de cliente de producción y el camino de render de React mientras controla la carga de trabajo y la cadencia de llegada.

## Consecuencias

La tasa de publicación de los objetos `ConversationSnapshot` en streaming está acotada por la tasa de pintado del navegador, así que React maneja como mucho un parcial acumulado que contiene todo el texto recibido por frame; los eventos estructurales aún pueden publicarse antes. La ingesta, la ordenación, el registro, la concatenación de cadenas y las actualizaciones del acumulador siguen corriendo por cada chunk crudo, así que esta decisión reduce la reconstrucción de instantáneas y el trabajo de React sin pretender resolver el costo del parseo del stream crudo.

Las lecturas y escrituras de layout horizontales para el resumen Think colapsado corren como mucho una vez cada tres frames, y cada actualización mueve el resumen directamente a la posición más reciente; React sigue haciendo commit de instantáneas acumuladas con normalidad, y el resumen vuelve a la primera línea en la finalización. Esta política visual local no cambia la inmediatez del scroll del cuerpo ni de las interacciones del usuario.

El carril de estrés de navegador sigue proporcionando una señal de capacidad de respuesta desde la aplicación real ensamblada y un punto de entrada para el perfilado visible, pero las diferencias de hardware y de programación lo hacen adecuado solo como evidencia explícita de rendimiento. Los tests enfocados deterministas protegen los recuentos de publicación, el contenido acumulado y el orden de preempción, mientras que los carriles de test por defecto permanecen rápidos.
