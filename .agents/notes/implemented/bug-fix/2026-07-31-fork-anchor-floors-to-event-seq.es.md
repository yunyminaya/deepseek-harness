# Agent Note: El suelo del ancla de fork cae a un seq de evento

Status: implemented

English | [中文](2026-07-31-fork-anchor-floors-to-event-seq.zh.md)

## Problema

El botón de fork en un mensaje de asistente detenido no hacía nada — sin sesión hija, sin error, sin reacción visible.

El nodo congelado detrás de ese mensaje no es un evento de log. Tanto la proyección viva como el replay de historial lo acuñan con un seq de orden-de-flujo de `turnEnd.seq - 0.9`, ubicándolo estrictamente tras cada evento del turno abortado y antes del siguiente, y la vista de chat entrega ese seq de nodo al punto de entrada de fork sin cambiarlo. `session.fork` acepta un entero no-negativo en el wire, así que un ancla fraccionaria se rechaza como invalid-params antes de que la petición alcance el host, y la llamada a fork de la entrada de chat traga los fallos. Nada distinguía el rechazo de un botón inerte.

La regla de corte del host nunca fue el obstáculo. Un turno abortado termina con un `turn/end` registrado con razón `aborted`, así que es un prefijo completado como cualquier otro y el ancla simplemente nunca llegó.

## Decisión

`SessionRuntime.fork` aplica suelo (floor) a `atSeq` antes del RPC. La convención de seq fraccionario pertenece a `dsh-client-runtime`, que la acuña en ambas proyecciones, la viva y la de replay, así que el mismo paquete la convierte de vuelta a un seq de evento real en el límite del wire en vez de que cada llamador de UI tenga que recordarlo. Las anclas enteras quedan sin afectar.

El suelo aterriza dentro del turno propio del ancla y no recorta hacia atrás: todo turno abre con `turn/start`, así que `turnEnd.seq - 1` no puede ser él mismo un `turn/end` de un turno anterior. La regla del host de primer-`turn/end`-en-o-después cierra entonces sobre el turno que el lector clickeó, igualando la semántica de turno-completo que el botón de fork a nivel de mensaje ya prometía para turnos completados.

La suite de fork de apiproxy fija la mitad host del contrato: un ancla con suelo dentro de un turno abortado corta a través de ese turno y siembra la hija con él.

## Alternativas consideradas

**Aceptar `atSeq` fraccionario en el wire.** Descartado porque el contrato del host es un seq de evento, no una posición en un continuo; la forma fraccionaria es una convención de renderizado de un cliente, y admitirla dejaría al `atSeq` solo entre los payloads que llevan seq en tomar no-enteros.

**Ocultar el botón de fork en mensajes interrumpidos.** Descartado porque forkear un turno que el lector detuvo deliberadamente es una de las razones más fuertes para forkear, y la capacidad funcionó lado-host todo el tiempo.

**Aplicar suelo en el adaptador `forkAt` de la entrada de chat.** Descartado porque `ui-conversation` consume la convención fraccionaria sin poseerla; cualquier segundo punto de entrada de fork tendría que redescubrir la misma conversión.

## Consecuencias

Forkear desde un turno detenido produce una hija sembrada a través del `turn/end` de ese turno. El texto parcial congelado se reconstruye desde eventos de chunk y nunca fue un `assistant/message`, así que queda fuera de la transcripción modelo de la hija exactamente como queda fuera de la de la fuente al resumir — la hija resume desde el mismo contexto que la fuente.

Los fallos de fork siguen silenciosos en la entrada de chat. Este bug sobrevivió porque ese call site descarta su rechazo; mostrar errores de fork en la UI es un cambio separado.
