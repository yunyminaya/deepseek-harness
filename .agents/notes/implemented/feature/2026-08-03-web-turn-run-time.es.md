# Agent Note: Tiempo de ejecución de turnos Web y chrome de tiempo revelado al hover

Status: implemented

[English](2026-08-03-web-turn-run-time.md) | [中文](2026-08-03-web-turn-run-time.zh.md) | Español

## Problema

El chat Web muestra cuándo llegó un mensaje pero no cuánto trabajó el agente en él. Los turnos largos no dan señal de progreso en vivo más allá de la etiqueta estática de actividad, y tras asentarse el turno el tiempo real no es recuperable desde la UI. Mientras tanto, la fila del reloj siempre visible añade ruido visual a cada mensaje.

## Decisión

El tiempo real del turno usa las marcas de tiempo registradas existentes `turn/start` y `turn/end`, sin eventos de sesión nuevos. El Session del cliente pliega cada par dentro de la ventana en `turnTimings`; el footer del assistant propietario de las acciones renderiza `endTime - startTime` como etiqueta localizada `Ran for {duration}` tras terminar el turno. El reloj `TurnStatus` en marcha usa el último timing sin fin, así que la recarga preserva el tiempo transcurrido, el steering no lo reinicia, y un reintento arranca desde su propia frontera registrada. Ambas lecturas usan el mismo formateador localizado y el suelo de segundos completos. El reloj aparece solo después de 15 segundos y queda oculto para la región activa, de modo que los lectores de pantalla anuncian el estado de actividad sin reproducir cada tick.

El chrome de tiempo (reloj y tiempo de ejecución) se revela al hover: los contenedores de mensaje se acogen con un atributo `data-time-hover-root`, y `MessageIconActions.module.css` desvanece la etiqueta de tiempo en el `:hover`/`:focus-within` del contenedor. La regla está acotada a `@media (hover: hover)`, así que los dispositivos táctiles conservan la etiqueta siempre visible; la opacidad (no el display) mantiene el layout estable. Los iconos de copiar/ramificar siguen siempre visibles.

## Alternativas consideradas

**Derivar el timing de los nodos de mensaje.** La marca de tiempo del usuario o steering más cercana está disponible en el transcript renderizado, pero mide mal los turnos de reintento y deja que el steering a mitad de turno reinicie el reloj en vivo. Los eventos de frontera de turno existentes proporcionan las marcas de tiempo autoritativas sin cambiar el formato del registro.

**Anclar el reloj en vivo al montaje del componente.** Más simple, pero una recarga a mitad de turno reiniciaría el reloj a cero y discreparía con la etiqueta final del footer. El tiempo de montaje queda solo como fallback cuando `turn/start` está fuera de la ventana cargada.

**Ocultar toda la fila de acciones hasta el hover.** Copiar y ramificar son affordances que vale la pena descubrir, y el mostrar/ocultar a nivel de fila arriesga desplazamiento de layout. Solo el texto pasivo de tiempo queda gated al hover.

## Consecuencias

La duración del turno es visible en vivo y tras el asentamiento sin eventos de sesión nuevos, y ambas lecturas comparten fronteras y formateo exactos del registro. La duración asentada incluye la actividad tras el último texto del assistant hasta `turn/end`; la etiqueta está ausente cuando `turn/start` queda fuera de la ventana cargada. El chrome de tiempo ya no compite con el contenido del mensaje en reposo, y el reloj que hace tick sigue siendo visual en lugar de anunciado repetidamente.
