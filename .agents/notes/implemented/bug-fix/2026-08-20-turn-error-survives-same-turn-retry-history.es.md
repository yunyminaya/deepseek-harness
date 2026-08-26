# Agent Note: Los errores terminales de turno sobreviven al historial de reintentos del mismo turno

Status: implemented

[English](2026-08-20-turn-error-survives-same-turn-retry-history.md) | Español

## Problema

La Definition `turn-error` de Web suprimía su nodo permanentemente en cuanto el turno dueño portaba cualquier evento `llm/retry`. Esa regla codificaba el modelo de reintento [de recuperación acotada de peticiones LLM](../architecture/2026-06-21-bounded-llm-request-recovery.es.md) publicado originalmente, donde un reintento cerraba el turno fallido y abría el siguiente turno numerado: un turno con historial de reintentos solo podía ser un fallo intermedio cuyos hechos ya vivían en la fila de reintento, y el fallo terminal agotado aterrizaba en un turno posterior sin eventos de reintento.

El agent loop reintenta desde entonces dentro del turno y del paso que fallan —el invariante de runtime de `llm-retry` exige `llm/retry` dentro de un turno y un paso abiertos, y sus pruebas de loop verifican que un turno recuperado mantiene un `step/start`. Bajo ese productor, «el turno es dueño del historial de reintentos» y «este error de `turn/end` es el fallo terminal agotado» siempre coinciden, de modo que la supresión ocultaba exactamente el fallo al que existía para diferir: agotar todos los reintentos transitorios dejaba la conversación con una fila neutra colapsada «Retried model request (N/N)», sin fila de error y con el composer re-habilitado. Los escenarios e2e en vivo no detectaron el hueco porque cubrían un fallo AUTH no reintentable (sin eventos de reintento, así que la fila se renderizaba) y un fallo transitorio que se recuperaba (un turno completado no deriva ningún fallo), nunca el agotamiento.

## Decisión

Eliminar la supresión. La Definition `turn-error` coincide solo con `turn/start` y con los `turn/end` con razón de error, y renderiza siempre que su turno registró un error terminal; la cadena de reintentos concluida se renderiza junto a ella a través del nodo separado `model-retry`. Sin estado oculto, sin rama de retracción: con reintentos en el mismo turno no hay orden de eventos en el que un error terminal renderizado sea luego superado, porque `turn/end` cierra el turno.

Las ventanas de historial parcial se comportan idénticamente por construcción —una ventana de cola que contiene solo el `turn/end` con razón de error deriva el mismo nodo que el historial completo, donde la regla antigua ocultaba uno y mostraba el otro según qué eventos de reintento incluyera la ventana.

## Pruebas

La suite de Definition acciona el ensamblador real a través de una cadena de reintentos del mismo turno que termina en un `turn/end` con razón de error y verifica que el nodo `turn-error` se materializa con su mensaje y su código —en historial completo, en una ventana solo de cola y tras anteponer la cadena anterior. Un escenario de composición Web sin clave agota una política de dos reintentos propiedad del escenario contra tres lanzamientos SERVER inyectados y fija la fila de error terminal junto a la fila de reintento asentada en el golden; el scaffold ganó una opción `replayRetryPolicy` para que el agotamiento se ejecute en milisegundos en lugar de los cinco intentos con backoff del valor compartido por defecto.

## Alternativas consideradas

**Restablecer `hidden` cuando llega el fallo terminal.** Rechazado: conserva una máquina de estados cuya única transición restante es la que causó el bug. Con reintentos en el mismo turno ninguna secuencia de eventos necesita la supresión en absoluto.

**Distinguir los errores `turn/end` intermedios de los terminales.** Rechazado: la distinción no existe en el log. Un turno termina una vez; una razón de error es siempre terminal para su turno.

## Consecuencias

La recuperación agotada deja ahora retroalimentación durable y reproducible: la fila roja terminal con el mensaje y el código seguros para pantalla, más la cadena de reintentos colapsada como contexto de recuperación. Los logs de sesión registrados bajo el modelo retirado de reintento con nuevo turno renderizarían una fila `turn-error` por turno fallido en la reproducción; la postura de formato pre-release lo acepta, y ningún productor de logs publicado ha emitido esa forma desde que aterrizaron los reintentos del mismo turno.
