# Agent Note: el límite end-seed del log

Status: implemented

[English](2026-07-30-session-end-seed-log-boundary.md) | Español

## Problema

Un plugin que posee un marcador de apertura/cierre autónomo en el log de la sesión no puede distinguir un marcador muerto de uno vivo. `compaction/start` … `compaction/end` es el caso publicado: al recoger un log cuyo último evento de compactación es un `compaction/start` sin cerrar, «el escritor anterior murió a mitad de la compactación» y «una compactación se está ejecutando ahora mismo» son historia almacenada byte a byte idéntica. El propietario debe o bien negarse a compactar un log que en realidad está libre (atascando la sesión), o bien proceder sobre uno que está genuinamente ocupado.

Nada en el log marcaba dónde terminaba la historia heredada. `session/created`, `session/disposed` y `session/flush` son señales de runtime de cordis, no eventos de log; `agent/session-start` es solo de emisión. `Session.firstLiveSeq` ya contenía exactamente la respuesta — el seq de la primera escritura propia de este ciclo de vida — pero solo en memoria, así que un consumidor que lee los bytes almacenados no podía verla.

El arreglo de fallos no cierra el hueco y no debe hacerlo: `interruptedTurnClosers` sintetiza los límites de turno, paso y herramienta porque el núcleo es dueño de ese vocabulario, y `compaction/*` pertenece al seam de compactación. Un pase de arreglo del núcleo que cerrara los marcadores de los plugins pondría la semántica de marcadores de cada plugin en el núcleo.

## Decisión

El constructor de `Session` añade el evento `session/end-seed`, exclusivo del log, inmediatamente después de una semilla de constructor suministrada explícitamente, incluida una vacía, como la primera escritura viva de la sesión sembrada en el seq que nombra `firstLiveSeq`. El evento es la proyección durable de ese campo: `firstLiveSeq` responde dónde empiezan las escrituras de este ciclo de vida para un consumer que tiene el objeto, mientras que `session/end-seed` responde a la misma pregunta para uno que solo tiene los bytes almacenados. Su payload está vacío — la posición y `time` llevan todo el significado — y no es un `SurfaceEventType`, así que no produce ningún mensaje y no puede perturbar la historia derivada. El marcador de seq 0 distingue una sesión reanudada vacía de una sesión realmente nueva, evitando que los valores predeterminados de sesión nueva se apliquen durante la reanudación.

Un propietario de marcadores lo lee posicionalmente: un marcador de apertura sin cerrar antes de `session/end-seed` tiene un seq menor, vino de la semilla del constructor y pertenece a un ciclo de vida que ha terminado. El núcleo escribe el límite y no lee nada de él; el vocabulario de cada marcador permanece en su plugin propietario, así que ningún helper de predicado del núcleo se publica sin un consumer que le dé forma.

El constructor es la ubicación porque es el único estrechamiento por el que pasa toda sesión sembrada. Todos los seis puntos de entrada llegan a él: `agents.resume()`, el arranque dirigido por configuración sobre un id persistido (`restoreOrCreateConfigured`), `sessions.fork()`, un hijo de fork de subagente, la ruta de prefijo vivo de `coordinator.adopt()` y un `sessions.create(id, {seed})` desnudo. Un límite escrito en la carga de persistencia se perdería ambas rutas de fork — y un hijo de fork que hereda un `compaction/start` abierto de un padre aún en ejecución es precisamente el caso que debe poder clasificarse. Un límite escrito al inicio del loop se perdería `fork()` y `adopt()`, y tendría que dispararse en `SessionStartSource: 'startup'`, que es lo que publica un hijo de fork, así que ese campo dejaría de discriminar.

Dos guardas mantienen el marcador preciso. Una semilla omitida no escribe nada porque la sesión es nueva. Una semilla que ya termina en uno no se vuelve a marcar, lo que hace idempotente la escritura. La idempotencia es estructural, no pulcritud: cada recogida de una sesión fría vinculada a un Agent pasa por `agentFor()`, y sin la guarda los controles repetidos harían crecer el log incluso cuando no realizan ningún trabajo. Las rutas de origen solo de inspección `session.history` y `session.fork` no crean este límite en el origen.

## La persistencia no necesita cambios

El añadido del constructor ocurre antes de `enter()`, así que la sesión no tiene adjunto de almacén: el marcador nunca se publica en `session/event`, exactamente como los eventos de semilla anteriores. En cambio, es parte del log que `initFor` captura como semilla de creación, y persiste por la ruta de semilla ordinaria — el `createCore` + `appendCore` de `onCreated`, o la escritura de sufijo de reclamo sin propietario. Un consumer que observa el firehose por tanto nunca ve el límite y debe leerlo del log.

Consecuencias para el seam: `load()` sigue siendo una lectura pura, sin subir la revisión, sin `commitRepair` en un log equilibrado y sin marca durable dejada por un `append` rechazado. **Adjuntar no es una lectura pura**, sin embargo — una recogida ahora escribe donde antes no se escribía nada, así que un disco de solo lectura o lleno falla en `session/created` en lugar de en el primer turno real. Ese es el único costo que añade esta ubicación, y es más estrecho que el de la versión de la ruta de carga (que fallaba la propia carga).

Un fallo antes de que la escritura de la semilla llegue al disco pierde el límite, y eso no cuesta nada: el lote pendiente se escribe en orden, así que un límite perdido significa que también se pierde todo evento posterior. La siguiente recogida lee los mismos bytes que la anterior, añade su propio límite y clasifica el marcador de forma idéntica. Los consumers en proceso deberían preferir `firstLiveSeq`, que es exacto antes de cualquier escritura.

## Alcance de la garantía

El predicado se cumple para un marcador que *esta* sesión heredó, no como señal de vivacidad sobre otros escritores. Una sesión viva concurrente puede tener un marcador abierto sobre la misma historia almacenada mientras su propio límite está en otro sitio. Un consumer que deba tolerar escritores concurrentes necesita una señal de vivacidad más allá del log y no puede omitirla apoyándose en este evento.

## Alternativas consideradas

**Un límite escrito por la ruta de carga en frío del coordinador de persistencia.** Una iteración anterior escribía un límite `session/resumed`; perdió porque no cubre ningún fork — el único caso en que el propietario del marcador heredado puede seguir en ejecución — y porque un marcador acuñado en la carga tenía que ser una escritura durable en una ruta de lectura, lo que repartía el costo por todo el seam: una subida de revisión en cada carga en frío, un lote `commitRepair` en un log equilibrado sin nada que reparar, un suelo de tiempo almacenado para mantener monótono el clamp, y una carga que fallaba contra un almacén de solo lectura.

**Un límite añadido al inicio del loop.** El loop llama a `resumeWith`, así que cubre las rutas de reanudación, pero se pierde `fork()` y `adopt()` por completo, y el evento tendría que dispararse en `'startup'` — el origen que publica un hijo de fork — así que `SessionStartSource` dejaría de discriminar. Además publica la sesión antes de añadir el marcador, así que un listener de `session/created` podría observar un log sembrado sin límite.

**Reutilizar `header.seedLength`.** Es el límite durable del *linaje de fork* y conserva deliberadamente el valor de fork original a través de una reanudación, donde la semilla del constructor es todo el log almacenado. Los dos hechos difieren y confundirlos perdería ambos.

**Que el arreglo de fallos cierre `compaction/*` junto con los límites de turno.** Rechazado: mueve la semántica de marcadores de cada plugin al pase de arreglo del núcleo, y el núcleo no puede saber qué debería registrar el cierre del marcador de otro paquete.

## Consecuencias

Comprado: un límite, escrito en un solo lugar, correcto para las seis rutas de inicio sembrado — incluido el hueco de fork que la versión de la capa de persistencia no podía alcanzar. Los paquetes de persistencia conservan una ruta de lectura pura. `firstLiveSeq` gana un gemelo durable en lugar de una segunda noción competidora del mismo límite.

Costo: el log de una sesión sembrada es un evento más largo, incluido un log reanudado vacío. Las expectativas de seq se mueven con ese límite. Dos actualizaciones son estructurales, no mecánicas: las pruebas de adopción de telemetría afirman que el límite SE exporta, porque es la escritura propia de este ciclo de vida, y el invariante de replay de la suite de propiedades es «semilla reproducida verbatim, más un límite exclusivo del log», con la idempotencia como propiedad propia.

`session/end-seed` se une al vocabulario en disco. Bajo la postura previa al lanzamiento (`SESSION_FORMAT_VERSION` fijado en `0`, sin promesa de compatibilidad), los logs más antiguos simplemente carecen de él, y un log sin límite clasifica correctamente que nada es historia de semilla de constructor.

La [decisión de compactación manual en cola](../feature/2026-07-30-queued-manual-compaction.es.md) ahora aporta el primer consumer. Su escaneo de cola encuentra de forma independiente el `compaction/start` sin cerrar y el end-seed más reciente, trata solo como vivo un inicio posterior a ese límite y limpia la traza de invariante en la misma transición de replay. El predicado permanece en el paquete de compactación en lugar de convertirse en un helper genérico del núcleo.
