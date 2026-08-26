# Agent Note: Uso de tokens proyectado y ocupación del contexto

Status: implemented

[English](2026-07-29-projected-token-usage-and-request-context.md) | [中文](2026-07-29-projected-token-usage-and-request-context.zh.md) | Español

## Problema

La línea de estadísticas de la Web derivaba los totales de tokens de los nodos de conversación cargados actualmente. Esa ventana está paginada, por lo que el desplazamiento cambiaba los totales, y la compactación reemplaza el contenido visible sin conservar la facturación que hay detrás. La facturación durable del provider necesita una fuente que sobreviva a ambas cosas.

La ocupación del contexto necesita un numerador y un denominador que ninguna superficie existente llevaba al navegador: el tamaño del prompt de la última petición y la capacidad de la ruta que usó.

## Decisión

Ambos valores son estado durable ordinario de la proyección de sesión. `@deepseek-ai/dsh-token-meter` registra dos unidades cuando `ctx.sessionProjections` está presente.

`tokenUsage` pliega el log durable completo en cubos de entrada sin caché, salida, lectura de caché y escritura de caché. Una muestra de uso de `assistant/chunk` sobrevive a una petición fallida posterior; un valor de uso de `assistant/message` para el mismo `(turn, step)` reemplaza la muestra anterior en lugar de contarla dos veces. El razonamiento sigue siendo una subdivisión de la salida. La compactación y el reemplazo de superficie no borran la facturación anterior.

`contextPressure` lleva un `pressureTokens` opcional — el tamaño de prompt más reciente reportado por el provider, que suma la entrada sin caché más las lecturas y escrituras de caché, excluyendo la salida — y un `contextWindow` opcional del registro `request/context` más reciente. Ninguno de los dos campos se sintetiza antes de que exista su fuente.

`request/context` es un evento de sesión nuevo, solo de log, que registra metadatos ligados al registro para la ruta a la que se resolvió una petición. AgentLoop lo añade dentro del paso junto a `request/header`, a partir de los metadatos de contexto que `prepareCall()` ahora devuelve junto con la config resuelta — la misma búsqueda ligada al registro que ya validaba el razonamiento, por lo que no ocurre una segunda resolución. Se omite cuando el provider, el modelo y la capacidad coinciden todos con el registro anterior. Una ruta cuyo adaptador no anuncia capacidad se registra con `contextWindow` ausente, borrando el denominador de una ruta anterior.

La capacidad queda deliberadamente fuera de `EpochHeader`. Ese tipo es el contrato de reconstrucción — aquello a partir de lo cual se construyó una petición — y `headerEquals` lo compara campo a campo para decidir si una instantánea es un cambio real. La capacidad es metadatos del adaptador que describen una ruta, por lo que colocarla ahí permitiría que un cambio de capacidad se hiciera pasar por un cambio del sobre de petición y la arrastraría al invariante de reconstrucción del loop.

Ambas unidades viajan en el ciclo de vida estándar de la proyección: líneas base de la cola del historial, tramas en vivo de `session/projection`, almacenamiento de cliente en el que gana el seq más alto, puntos de control JSON, recuperación de caché y descarga de unidad. No hay campo de historial específico de tokens, ni trama mux, ni proyector, ni contador de revisión, ni valla de cliente.

La `StatsLine` de la Web lee ambos a través del seat estándar `useProjection`. Los nodos de la ventana siguen suministrando los recuentos de turno y paso además de los tiempos de pared de LLM y de herramientas — esos responden a «qué hay en pantalla» y están correctamente acotados a la ventana. Los grupos durables de tokens y de contexto permanecen cuando la compactación no deja ningún paso de asistente visible. Las escrituras de caché cuentan en la entrada facturada y en el denominador de aciertos de caché. Un despliegue sin token-meter elimina los grupos de tokens; la ocupación permanece oculta hasta que se conocen tanto la presión como la capacidad.

## La ocupación del contexto es aproximada, y esa es la decisión

`pressureTokens` y `contextWindow` son campos independientes en los que gana el último, no una observación atómica. Cambiar de modelo empareja una capacidad nueva con la presión de la ruta anterior hasta que la siguiente petición reporta el uso, y el numerador describe la última petición en lugar de la superficie tal como está actualmente.

Esto se aceptó deliberadamente. Un porcentaje de ocupación es una cifra de referencia orientada al usuario: nada en el harness toma decisiones a partir de ella, y la compactación lee `measure()` directamente en su lugar. La línea de estado de la TUI siempre ha calculado la ocupación así, dividiendo un total de `measure()` por una capacidad resuelta por separado para el modelo seleccionado — por lo que una variante atómica aquí habría sido el valor atípico, no la norma.

La no atomicidad es deliberada, no un defecto. Un consumidor que necesite de verdad una cifra exacta de la misma frontera debería llamar a `ctx.tokenMeter.measure()` en su propia frontera de petición, donde ambos valores están disponibles juntos, en lugar de leer esta proyección.

## Alternativas consideradas

**Una instantánea atómica de frontera de petición entregada como trama mux transitoria (implementada y luego rechazada).** Una revisión anterior emitía `session/model-request`: una trama no reproducible que llevaba `contextTokens` y `contextWindow` medidos en la misma frontera de `agent/model-request`. Ser la única clase no reproducible del stream mux es lo que la rompió. Host y mux son streams SSE independientes sin orden entre streams, por lo que una petición emitida antes de una eliminación podía llegar después de `host/session-removed` y revivir la telemetría de una sesión muerta, mientras que una petición legítima para un ciclo de vida nuevo que reutilizara el mismo id podía quedar bloqueada por una eliminación tardía. `session/subscribed` no es prueba de ciclo de vida — dice que una cola empezó a suscribirse a un id, no que una sesión nueva en memoria reemplazó a una anterior — y `lastSeq` es una marca de agua durable que dos ciclos de vida pueden compartir. Una corrección correcta exigía una generación de ciclo de vida monótona en la trama, en la suscripción y en la eliminación, además de una comparación de marca de agua en el cliente.

Ese costo compró una visualización peor: la ocupación quedaba en blanco tras cada reconexión y nunca se movía mientras una conversación crecía. También convertía a ApiProxy en un punto de medición que llama al `measure()` O(superficie) en cada petición, y expresaba el estado de reconexión a través de un error de apertura `cancelled` sintético que la UI tenía que tratar de forma especial.

**Plegar la ventana de nodos cargados en React.** No puede sobrevivir a la paginación ni a la compactación, y hace que un paquete de presentación reconstruya la semántica del log.

**Publicar el uso solo con los mensajes finales de asistente.** Una petición que reporta un chunk de uso y luego falla perdería su facturación.

**Resolver la capacidad dentro de token-meter.** El paquete se documenta como independiente del enrutamiento de modelos y por lo demás es un lector puro que nunca añade al log. AgentLoop ya posee los metadatos resueltos donde se escribe la cabecera.

**Extender el RPC `session.models` con la capacidad.** El handler ya la resuelve y la descarta, por lo que el campo sería casi gratis — pero `StatsLine` vive en `ui-conversation` mientras que el directorio de modelos vive en `ui-model-selection`, y `ui-conversation` no puede depender de `ui-model-selection`. Entregarlo habría exigido una segunda entrada del dock que dividiera una fila de texto entre dos plugins, o una escritura de almacén entre plugins.

**Añadir un círculo de contexto junto al selector de modelos.** Esa ubicación sugiere estado del modelo seleccionado. La línea de estadísticas lleva la cifra sin una UI ni una vía de datos duplicadas.

## Consecuencias

Los totales de tokens permanecen estables a través de la paginación, la compactación, la reproducción, el reinicio y la reconexión, porque son estado durable ordinario de la proyección recuperado por las vías genéricas. La carrera de reordenamiento entre streams desaparece por construcción en lugar de quedar vallada.

La ocupación es aproximada en las formas documentadas arriba. Está disponible inmediatamente después de la restauración o la reconexión, ya que ambos campos son durables, al costo de describir la última petición registrada en lugar de una frontera actual exacta.

Cada log de sesión gana un registro pequeño `request/context` por ruta o por cambio de capacidad anunciada. La proyección de token-meter es la propietaria canónica de la semántica durable de uso de la proyección de sesión; la TUI conserva su mapa vivo por paso porque no monta el seam genérico de proyección, y el fixture de navegador independiente refleja la unidad. ApiProxy no lleva código específico de tokens, no posee caché de métricas por sesión y no realiza mediciones. El navegador conserva dos valores genéricos de proyección y ninguna telemetría local a la conexión, y los deltas de texto en streaming siguen sin obligar a la línea de estadísticas a recalcular.
