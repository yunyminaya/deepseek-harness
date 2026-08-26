# Agent Note: El estado de replay se alinea con el contenido ensamblado por construcción

Status: implemented

[English](2026-08-15-max-token-replay-state-alignment.md) | Español

## Problema

pi-ai registraba un blob de replay opaco por respuesta, proyectado a partir del mensaje nativo del provider, mientras que `BlockAssembler.blocks()` descartaba por separado las llamadas de herramienta de una respuesta `max-tokens`, porque una llamada truncada no es segura de ejecutar. El mensaje durable del assistant almacenaba, por tanto, contenido transformado junto a metadatos que describían la lista de bloques nativa sin transformar. La siguiente petición fallaba durante la reconstrucción del historial con `INVALID_REPLAY_STATE: block count does not match assistant content`, y como el desajuste ya estaba en disco, toda petición posterior de esa sesión fallaba del mismo modo: la sesión quedaba bloqueada permanentemente. La causa raíz es estructural: dos representaciones de una misma respuesta se instantaneaban en puntos distintos de la canalización, y su alineación de índices solo se mantenía mediante un error duro en tiempo de lectura.

## Decisión

Dos cambios, uno por cada lado del límite durable.

**Lado de escritura: una sola decisión de conservar/descartar.** El `replayState` del chunk final se convierte en un `ReplayEnvelope` tipado: una mitad `response` opaca más entradas opacas opcionales por bloque, alineadas con la secuencia de bloques emitida. `BlockAssembler` calcula su decisión de conservar/descartar una sola vez y la aplica a bloques y entradas del envelope a la vez, de modo que cualquier transformación que realice el ensamblado —el descarte actual de llamadas de herramienta por max-tokens o uno futuro— poda los metadatos correspondientes por construcción. Los bloques conservados mantienen sus entradas, así que una respuesta truncada conserva las firmas del razonamiento y del texto que retuvo. Un envelope cuyas entradas no coinciden con el número de bloques emitidos se descarta entero (un adapter que emite mal no debe publicar metadatos mal atribuidos). pi-ai divide su antiguo estado plano en una mitad de respuesta versión 2 y entradas de firma por bloque.

**Lado de lectura: el contenido durable es la autoridad.** `toPiAssistant` trata el estado de replay como metadatos de fidelidad, no como una entrada que soporta carga: cualquier estado que la build de lectura no pueda usar —el kind de otro adapter, otra versión (incluida la forma plana versión 1 ya en disco), metadatos malformados o una forma de bloque que ya no coincide con el contenido— degrada ese único mensaje a la conversión existente neutral respecto al provider y notifica el diagnóstico `INVALID_REPLAY_STATE` a través del hook `onReplayDegrade` del plugin (un aviso del logger). La petición continúa. Esto es lo que permite que las sesiones envenenadas antes de este cambio sigan adelante en lugar de fallar para siempre, y acota toda fuente futura de divergencia a una pérdida de fidelidad en un único mensaje.

## Verificación

Las pruebas unitarias del ensamblador demuestran la poda, el descarte por desalineación y el paso directo de envelopes sin transformar y sin entradas por bloque. Las pruebas unitarias de pi-ai demuestran el viaje de ida y vuelta del envelope versión 2 y que todo caso de estado inválido que antes lanzaba una excepción ahora degrada a la conversión externa con el diagnóstico. Una regresión del agent loop lleva una respuesta truncada de texto más llamada de herramienta a través de la persistencia y muestra la petición siguiente portando el envelope podado. Las pruebas de composición real sin clave arrancan `dsh-llm-pi-ai` a través del Loader y demuestran una continuación nativa sin `tool_calls` tras el truncamiento, y una continuación satisfactoria sobre un mensaje plano heredado cuyo número de bloques ya no coincide. El escenario de instantánea sin clave `max-tokens-continue` fija el registro durable de la aplicación ensamblada —turno truncado, envelope podado en el mensaje almacenado, turno continuado— a través de la ruta real del subproceso ACP.

## Alternativas consideradas

**Suprimir todo el estado de replay cuando el ensamblado descarta una llamada de herramienta.** Funciona para la única transformación actual, pero vuelve a derivar la condición de descarte junto a `blocks()` (las dos se desvían en silencio), descarta firmas válidas de los bloques conservados y deja la divergencia en tiempo de lectura —sobre todo las sesiones heredadas en disco— como error duro.

**Conservar el estado y relajar la validación del número de bloques de pi-ai para adjuntar lo que encaje.** Rechazado: firmas alineadas por índice adjuntadas a una lista de bloques distinta presentarían al provider un historial nativo falso. La degradación no adjunta nada.

**Enseñar a cada adapter a reescribir su estado después del ensamblado.** Rechazado como obligación del adapter con un blob opaco; el envelope mueve exactamente la estructura necesaria —y nada más— al vocabulario compartido, y la decisión única del ensamblador hace la reescritura mecánicamente.

## Consecuencias

Continuar tras una respuesta max-tokens que incluía una llamada de herramienta funciona, conserva las firmas nativas de los bloques mantenidos y se reproduce como un mensaje pi-ai nativo. Las sesiones registradas antes de este cambio reproducen sus mensajes de assistant afectados como contenido neutral respecto al provider (con un diagnóstico) en lugar de fallar el turno; los valores de `replayState` en disco cambiaron de forma bajo la postura pre-release de no compatibilidad, y la antigua forma plana se gestiona por la misma ruta de degradación. Esto sustituye la regla de error duro en tiempo de lectura de la [decisión de adaptadores enrutados por provider](../architecture/2026-07-14-provider-routed-llm-adapters.es.md) para el estado inutilizable; la validación en sí no cambia y sigue precediendo a cualquier reconstrucción nativa.
