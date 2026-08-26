# Agent Note: encolado de seguimientos y límites de ejecuciones propias

Status: implemented

[English](2026-07-30-followup-enqueue-and-owned-runs.md) | Español

## Problema

`Agent.followup()` identifica y pone en cola un mensaje de usuario, pero un seguimiento no es dueño de la actividad que lo sigue. El steering (direccionamiento), el contexto inyectado, las continuaciones de herramientas, la recuperación y los mensajes encolados posteriores pueden contribuir antes de que el agent (agente) vuelva a quedar inactivo. Un `MessageId` puede por tanto probar la admisión en el inbox, pero no puede identificar qué mensaje de asistente o qué `turn/end` es el resultado de esa entrada.

La [decisión one-send-one-turn](../simplification/2026-07-17-one-send-one-turn.es.md) ya rechaza un handle de finalización por envío en la API del núcleo. Las capas de protocolo y SDK que emparejan una solicitud de prompt con un resultado de turno fabrican esa relación ausente aguas abajo. El emparejamiento se vuelve ambiguo en cuanto la actividad admite más entrada, y expone la mecánica de los turnos como si fuera un resultado a nivel de prompt.

## Decisión

Mantener `Agent.followup(message): void` como una operación solo de encolado. `Agent.whenIdle()` y `agent/status` siguen siendo observaciones del ciclo de vida del agent completo; ninguno de los dos cierra un mensaje individual. La durabilidad del inbox registra el mensaje identificado y su admisión o cancelación, sin asignarle la salida posterior.

El protocolo SDK de bajo nivel responde `session/prompt` en cuanto el encolado tiene éxito con `{ messageId }`. Transmite hechos duraderos por `session.event`, publica transiciones del agent completo por `session.status` y no tiene `session.finished`. Un cliente de bajo nivel puede observar ese recibo y la inactividad posterior, pero no recibe ningún resultado de prompt.

Las API de automatización de alto nivel devuelven un `RunResult` solo cuando poseen explícitamente un intervalo de actividad. Los métodos `run()` del SDK de TypeScript y de Python recogen desde el recibo durable del inbox del mensaje enviado hasta el siguiente `idle` del agent completo; su respuesta final es el último mensaje de asistente confirmado en ese intervalo, no una respuesta atribuida causalmente al prompt enviado. El SDK de Python también informa el tipo de razón del último turno raíz como el [`finish_reason`](../bug-fix/2026-08-11-owned-run-finish-reason.es.md) a nivel de ejecución, sin atribuirlo al prompt enviado. El CLI (interfaz de línea de comandos) de un solo uso posee el intervalo análogo de inactivo a inactivo. Una ejecución aislada de un agent hijo puede informar un resultado porque su llamador posee el ciclo de vida completo del hijo y cualquier steering pertenece a esa ejecución.

ACP (Agent Client Protocol) debe devolver un `stopReason` de protocolo. Su bridge serializa un prompt en vuelo por sesión ACP, espera la inactividad del agent completo y, por lo demás, informa el `end_turn` genérico. Los finales por límite de tokens no se atribuyen al prompt: se cierran como `end_turn`. Un error de modelo en el turno correlacionado con el prompt sí rechaza el prompt de inmediato (el error se atribuye a su turno propietario), y un slot sin turno (la admisión descartó el prompt) se cierra como `cancelled` en la inactividad, junto con la cancelación ACP explícita o la disposición.

La continuación de objetivos conserva `MessageId` solo para reconocer su mensaje de objetivo durable encolado y admitido. Avanza desde el estado durable del objetivo en la inactividad del agent completo, sin mapear el mensaje a un resultado de turno.

## Alternativas consideradas

**Mapear `MessageId` al turno que lo admite.** Un turno puede consumir steering y contexto inyectado y puede continuar por varios pasos de modelo/herramienta. El mapeo identifica la admisión, no la propiedad causal de la salida resultante o de la razón de parada.

**Devolver un handle de finalización por seguimiento.** Un handle implicaría un límite de resultado que el ciclo de vida compartido del agent no tiene. O bien omitiría el trabajo que influyó en la actividad, o bien absorbería en silencio entrada posterior no relacionada.

**Usar el último `turn/end` observado antes de la inactividad.** Es una observación útil a nivel de ejecución para un intervalo poseído explícitamente, pero nombrarlo como el resultado del mensaje enviado recrea la afirmación causal falsa.

## Verificación

- Las pruebas del agent y del inbox fijan el seguimiento solo-encolado, la admisión o cancelación durable y la observación de inactividad del agent completo.
- Las pruebas del protocolo SDK, del SDK de TypeScript y del SDK de Python fijan el recibo `{ messageId }`, `session.status`, la ausencia de `session.finished` y la recolección de `RunResult` de recibo-a-inactividad sin `status` o `reason` a nivel de prompt; las pruebas del SDK de Python fijan por separado su observación de `finish_reason` a nivel de ejecución.
- Las pruebas de ACP, del CLI de un solo uso, de la continuación de objetivos y de subagentes fijan la propiedad de actividad distinta que posee cada integración.
- Las pruebas de Consumer fijan que ninguna integración de producción deriva un resultado de seguimiento correlacionando `MessageId` con `turn/end`.

## Consecuencias

Un intervalo de actividad propio puede incluir steering, contexto inyectado u otro trabajo enviado antes de la inactividad, así que su respuesta final, su razón de finalización y sus eventos son deliberadamente más amplios que el mensaje iniciador. Las clasificaciones a nivel de prompt de error de modelo y de límite de tokens siguen ausentes de los resultados del SDK y de ACP; los llamadores pueden inspeccionar hechos a nivel de ejecución o duraderos sin reclamar atribución causal. La automatización concurrente en una misma sesión exige una política explícita de serialización o propiedad en lugar de un resultado implícito por prompt.
