# Agent Note: El transcript humano proyecta eventos append-origin

[English](2026-07-29-human-transcript-append-origin.md) | [中文](2026-07-29-human-transcript-append-origin.zh.md) | Español

Estado: implementado

## Problema

El terminal y el gateway de historial del host trataban ambos la superficie visible para el modelo como el transcript (transcripción) humano. Una compactación con éxito sustituye un rango de la superficie por un único nodo de checkpoint, así que en el instante en que aterrizaba ese reemplazo el terminal descartaba todos los mensajes que quedaban sombreados — conversación que el usuario ya había leído — y volvía a ejecutar esa reconstrucción destructiva en cualquier reemplazo posterior. La misma confusión llegaba a la paginación: `maxMessages` contaba todos los `user/message` y `assistant/message` de la ventana, así que una copia de reemplazo solo para el modelo consumía una ranura de página que el humano nunca había llenado, y el corte podía caer entre el evento `compaction/summary` de solo log de una compactación y el reemplazo que lo cita.

Nada se perdió del log. `Session.events` seguía conteniendo todos los mensajes originales y los resultados completos de las herramientas; la superficie solo decide qué se le envía al modelo después. El defecto estaba íntegramente en la proyección.

## Decisión

Las proyecciones de modelo y de humano son independientes, y el marcador propio del evento decide a cuál pertenece cada evento. `dsh-session` exporta la división de marcadores `isAppendSurfaceEvent(event)` y `isReplacementSurfaceEvent(event)` sobre las dos variantes de `SurfaceOp`, desde el módulo `surface` seguro para el navegador. Los eventos append-origin son la fuente duradera de un transcript; las copias de reemplazo se quedan solo para el modelo. Todo lo que debe enviar exactamente lo que ve el modelo — `deriveMessages`, la contabilidad de tokens, los backends de compactación, el emparejamiento de herramientas, la vigencia del contexto inyectado, la proyección de referencias entre sesiones — sigue leyendo `session.surface`.

El terminal reproduce el transcript a partir de los eventos de superficie append-origin y mantiene emparejadas las tarjetas de herramienta de un paso sombreado mediante `transcriptToolCallIds`, que lee el `assistant/message` append-origin en lugar de la pertenencia a la superficie. Una compactación aterrizada contribuye una fila atenuada `… earlier context was compacted …` en su propia posición de log: el marcador informa de dónde dejó el modelo de ver ese historial en lugar de borrarlo. El payload del checkpoint enmarcado nunca se renderiza, y ambas rutas clasifican un evento de superficie con el mismo marcador, así que una compactación que llega en vivo y el mismo log reproducido tras reanudar producen el mismo transcript. Solo la reproducción vuelve a derivar el emparejamiento de `tool/call`: un evento de llamada no lleva marcador propio y hereda la pertenencia del `assistant/message` que la anunció, que el listener en vivo ya ha renderizado necesariamente.

Un checkpoint se reconoce mediante el contrato propio del seam de compactación — `isCompactCheckpointSource`, el marcador independiente del backend que `CompactionEngine` exige en el mensaje de usuario de reemplazo — así que el terminal depende del vocabulario declarado, no de la forma del reemplazo. `dsh-session-reference` ya consume ese predicado para proyectar el log de otra sesión; es la misma pregunta formulada por un lector distinto. Otros reemplazos son silenciosos: un `tool/result` podado y un `assistant/message` regenerado reescriben un nodo para el modelo y no marcan ningún límite en la conversación.

`session.history` solo cuenta los mensajes append-origin para `maxMessages`. Cada página sigue siendo un rango contiguo de eventos brutos, así que el evento `compaction/summary` de una compactación permanece en la página del reemplazo que lo cita.

No cambió ningún evento persistido, envoltura RPC, transacción de compactación ni superficie visible para el modelo, y no se requiere migración.

## Diferido

El cliente de navegador se corrige por separado, en [la nota de proyección del transcript web](2026-07-30-web-transcript-log-ordered-projection.es.md): proyecta el mismo transcript append-origin en orden de log y renderiza un componente de marcador, y cierra el agujero de paginación que este cambio abrió — porque `session.history` ya no gasta cuota en el checkpoint, nunca corta en el checkpoint y sus eventos fuente citados como unidad, así que una página puede llevar un checkpoint que cita un `surfaceOp.start` fuera de la ventana, que el plegado de superficie del navegador rechazaba. Ese agujero es anterior a este cambio (el conteo ya podía pasarse de un checkpoint hacia el rango que sombrea), pero cuando el checkpoint era el mensaje contado más antiguo, la antigua regla de paginación hacía que la página incluyera por casualidad todo el rango sombreado.

La [decisión archivada de progreso de compactación en vivo del terminal](../../archived/feature/2026-07-30-compaction-progress-visibility.md) usa eventos de corchete independientes para conducir el indicador existente de una celda. No cambia el marcador de finalización que aquí se posee ni añade escala: los `sourceEventSeqs` del checkpoint siguen disponibles para un conteo o rango justificado por separado. El progreso no necesita por tanto cambios en el contenido del marcador ni una extracción previa de `renderReplacement(event)`.

## Alternativas consideradas

**Reconocer un checkpoint por su forma (un `user/message` de reemplazo).** Rechazada: lee una coincidencia de los productores actuales en lugar de un contrato declarado, y cualquier productor futuro que sustituya un rango con un mensaje de usuario heredaría silenciosamente el marcador de compactación. El seam ya publica `COMPACT_CHECKPOINT_SOURCE` precisamente para que los consumidores reconozcan un checkpoint con independencia del backend.

**Seguir renderizando el checkpoint como tarjeta de contexto inyectado.** Rechazada: el checkpoint enmarcado es una envoltura de instrucciones escrita para el modelo, no contenido de conversación humana. Mostrarlo mientras se oculta el historial que reemplaza invierte lo que el lector necesita.

**Persistir un segundo transcript de visualización.** Rechazada: el log de solo append ya contiene el material fuente autoritativo, así que un registro paralelo no aporta nada y añade trabajo de migración y consistencia.

**Derivar el marcador del corchete `compaction/*` en lugar del checkpoint.** Rechazada para el transcript: el corchete es un par de marcadores de instante alrededor de una operación, mientras que el transcript necesita la posición en la que la superficie cambió de verdad. El corchete es la fuente correcta para el progreso y la duración, que este cambio no renderiza.

**Clasificar los eventos replegando el log, como hace `session-query` para la búsqueda (`current` / `shadowed` / `log-only`).** Rechazada: un plegado responde a una pregunta de todo el log, mientras que una proyección plantea una pregunta por evento que el marcador propio del evento ya responde en tiempo constante.

## Consecuencias

La compactación ya no borra el historial del terminal; una sesión compactada varias veces muestra un marcador por cada compactación aterrizada, en orden de log. Las páginas de paginación pueden llevar más eventos brutos que antes, porque la cuota se gasta solo en mensajes que un humano o un modelo produjeron de verdad.

`rebuildTranscript` ahora materializa un componente por evento append-origin de todo el log, y se ejecuta al montar, ante un cambio de esquema de color del terminal y en cada conmutación de razonamiento. La compactación solía acotar ese trabajo precisamente para las sesiones largas a las que sirve la compactación, así que el coste ahora crece con la longitud de la sesión en lugar de con la superficie. Ese es el intercambio que la corrección existe para hacer — el historial preservado es el punto — pero una estrategia de ventanas o de reutilización corresponde a quien primero mida una reconstrucción lenta, no a un perfilador posterior que se pregunte por qué creció el trabajo.

`dsh-tui` gana una dependencia del seam `dsh-compaction` por un predicado puro, reflejando el uso existente de `dsh-session-reference`. El terminal sigue sin necesitar ningún backend de compactación en tiempo de ejecución.

Dos comportamientos cambiaron junto con sus pruebas. La prueba de terminal de reemplazo de superficie antes fijaba el borrado ("hides shadowed tool calls") y ahora fija la preservación más exactamente un marcador, incluyendo que una copia de resultado podada, un mensaje de asistente regenerado y el reemplazo de un plugin ajeno no renderizan nada. El escenario de instantánea de compactación escribía una fuente `agent-instructions` mientras afirmaba fijar la compactación; ahora escribe una fuente de checkpoint real, y sus tres fixtures (datos de prueba) se vuelven a grabar para mostrar el prompt preservado, la tarjeta de herramienta completa y el marcador.

La equivalencia en vivo/reproducción anterior está fijada por fixtures, no solo afirmada aquí: `surface-replayed-compaction` se monta con el reemplazo ya almacenado y graba de forma idéntica byte a byte a la `surface-after-compaction-wide` de la ruta en vivo. Cambiar cualquiera de las dos rutas rompe esa igualdad, que es el punto — la proyección de reanudación es lo que falló para los usuarios, y los dos fixtures deben moverse juntos.
