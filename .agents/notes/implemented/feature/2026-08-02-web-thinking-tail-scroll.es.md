# Agent Note: Desplazamiento de la cola de razonamiento en Web — el razonamiento plegado sigue la salida en vivo

Status: implemented

[English](2026-08-02-web-thinking-tail-scroll.md) | [中文](2026-08-02-web-thinking-tail-scroll.zh.md) | Español

## Problema

La fila Think de Web renderizaba la primera línea de razonamiento como su resumen plegado tanto para bloques asentados como en streaming. Una vez que esa primera línea existía, cada delta posterior de razonamiento cambiaba solo texto de cuerpo oculto. Un modelo rápido parecía por tanto estacionario mientras pensaba, y el usuario tenía que expandir toda la cadena de pensamiento para verificar que la salida seguía moviéndose. El backlog de producto ya pedía «thinking: actualizaciones de cadena de pensamiento en desplazamiento, expandible»; la fila actual satisfacía solo la segunda mitad.

## Decisión

Solo una fila Think plegada cuyo bloque de razonamiento es la cola activa de streaming sigue la salida en vivo. Su resumen es la última línea no vacía en lugar de la primera línea asentada, y el elemento de resumen de una línea existente se convierte en un scrollport horizontal programático fijado en `scrollWidth - clientWidth` después de cada actualización de texto. La asignación directa de `scrollLeft` sigue deliberadamente los deltas reales sin inventar una velocidad de marquee independiente: los tokens rápidos se mueven rápido, un modelo en pausa se detiene y el texto corto permanece quieto porque el rango de desplazamiento es cero.

El comportamiento es propiedad de los componentes de presentación existentes. `AssistantMarkdown` elige la última línea solo mientras la fila Think está en ejecución; `ToolRow` ya posee el estado plegado/abierto y por tanto posee si su resumen debe seguir el extremo en línea. No cambia ninguna sesión, wire, evento duradero ni contrato visible para el modelo. Expandir elimina el resumen plegado y renderiza el cuerpo completo del razonamiento en el flujo ordinario de la página. Cuando la fila se asienta, restaura la primera línea estable y reinicia el resumen al borde izquierdo. Otras sumarías de herramientas y filas Think asentadas conservan su comportamiento de elipsis existente.

## Alternativas consideradas

**Animar un marquee CSS independiente del streaming.** Rechazada: seguiría moviéndose durante las paradas del provider y haría parecer rápido a un modelo lento, lo que rompe la señal de rendimiento que la interacción existe para exponer.

**Mostrar siempre un sufijo fijo de la cadena de razonamiento completa.** Rechazada: el corte por caracteres puede partir una palabra o un grafema, descarta el comienzo de la línea actual antes de que el desbordamiento lo requiera realmente, y salta en lugar de moverse con cada delta.

**Auto-desplazar el cuerpo de razonamiento expandido o la página de la conversación.** Rechazada: el contenido expandido es una superficie de lectura. Forzarlo a seguir lucharía contra un usuario que se desplaza hacia atrás; el seguidor pertenece solo al resumen plegado de una línea.

## Consecuencias

La fila plegada ahora comunica la cadencia del provider a través del movimiento del contenido además del barrido existente, mientras el transcript asentado permanece estable byte por byte. La actualización de desplazamiento corre solo en los renders de React que el acumulador de streaming ya causa; no añade temporizador, bucle de animación, suscripción, estado duradero ni tráfico de transporte. Una línea de razonamiento actual larga conserva su texto DOM completo y recorta programáticamente el prefijo ya desbordado, así que la expansión sigue revelando el bloque completo y la tecnología de asistencia lee el mismo texto de resumen actual.

## Pruebas

`packages/client/ui-conversation/tests/reasoning-row.client.spec.tsx` fija la selección de la última línea, la posición de desplazamiento calculada del borde derecho y el reinicio al asentarse a la primera línea y `scrollLeft = 0`. El escenario Chromium ensamblado sin clave de `apps/web/tests/lifecycle-chrome.e2e.ts` reproduce fragmentos de razonamiento grabados reales a un ritmo observable, estrecha el viewport hasta que el resumen se desborda y afirma que la fila Think plegada en vivo alcanza su extensión real de desplazamiento del navegador. Su golden de replay asentado permanece sin cambios, lo que prueba que el contrato histórico del resumen sigue estable.
