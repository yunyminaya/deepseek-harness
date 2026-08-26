# Agent Note: Tareas de subagente en segundo plano

Status: implemented

[English](2026-07-08-background-subagent-tasks.md) | [中文](2026-07-08-background-subagent-tasks.zh.md) | Español

## Problema

El [seam de subagent](2026-06-21-subagent-capability-seam.es.md) devuelve un `SubagentRun`, pero la herramienta orientada al modelo originalmente recolectaba cada ejecución de forma síncrona. Las delegaciones independientes y lentas, por tanto, mantenían abierta la llamada del padre o corrían en serie.

Los subagentes necesitan el mismo comportamiento de arranque, recolección, listado, parada, titularidad, notificación y limpieza que las demás herramientas de larga duración, sin adoptar semánticas de flujo de proceso. La sesión hija sigue siendo la traza detallada; el padre necesita la respuesta final o el detalle seguro del fallo más el estado de la tarea. Un hijo en segundo plano además sobrevive a la llamada de herramienta que lo inició, así que sus contratos de cancelación y de disposición por el dueño deben ser explícitos.

## Decisión

Cada instancia de `dsh-tool-subagent` puede exponer `run_in_background`, controlado por `enableRunInBackground` y habilitado por defecto. Una instancia deshabilitada omite el parámetro y rechaza en la ejecución un argumento forzado de segundo plano. La selección del provider sigue siendo configuración de despliegue, así que una instancia sigue registrando una herramienta con nombre distintivo para un provider.

Los subagentes en segundo plano usan el [runtime genérico de tareas en segundo plano](../architecture/2026-06-20-generic-long-running-tool-runtime.es.md). La recolección, el listado, la cancelación, los avisos de finalización y la guía de prompt provienen de `job_output`, `job_list` y `job_kill`; no hay herramientas acompañantes específicas de subagente.

Las llamadas en primer plano retienen su contrato síncrono: esperan el arranque del provider y `run.result`, devuelven texto final solo para `completed`, mapean los demás motivos terminales a un resultado de herramienta con error con el diagnóstico seguro opcional descrito por la [decisión de permisos no interactivos](2026-08-15-product-subagent-noninteractive-permissions.es.md), y siempre disponen la ejecución antes de devolver.

Para una llamada en segundo plano, la herramienta valida al padre y rechaza una señal de ejecución ya abortada antes de llamar a `ctx.jobs.start()`. El runtime de tareas hace el preflight de la API de control y de la limpieza del dueño antes de invocar el arrancador del productor. Ese arrancador crea un `AbortController` independiente y comienza `ctx.subagents.start()`; tras devolverse el id, la señal de la llamada de herramienta ya no es dueña del hijo.

El registro de la tarea mapea el seam de subagent como sigue:

- `kind` es `subagent`, `label` es la descripción suministrada por el modelo y `owner` es el agent padre.
- `cancel(reason?)` aborta el controlador propiedad de la tarea. La misma señal cubre el arranque pendiente del provider y el trabajo restante de la ejecución publicada.
- `done` espera el arranque del provider, el resultado del hijo y `run.dispose()`. Las ejecuciones completadas devuelven texto final, las abortadas pasan a `killed` y los demás motivos de parada pasan a `failed` con el diagnóstico del provider cuando existe. Los fallos de arranque, resultado y disposición se convierten en resultados fallidos en lugar de promesas de tarea rechazadas.
- `readOutput` está ausente. Mientras vive, `job_output` devuelve solo estado; tras el asentamiento, devuelve la salida final de forma idempotente. La actividad intermedia del hijo permanece en la sesión hija.

## Ciclo de vida

Un subagente en segundo plano pertenece a su agent padre y no es durable tras el cierre del dueño. El runtime de tareas ancla la limpieza al ámbito del dueño exacto. La disposición del agent cancela la tarea y espera la reversión del arranque o la disposición del hijo antes de que `AgentHandle.dispose()` se resuelva, evitando agents y sesiones hijas con fuga.

Los avisos de finalización apuntan al dueño exacto capturado en el arranque. Si el desmontaje del dueño ya dispuso el destino de inyección, el aviso se descarta; la garantía del ciclo de vida es la limpieza, no la notificación.

## Guía para el modelo

El prompt genérico de tareas enseña el hábito compartido: retener ids, continuar el trabajo independiente en lugar de sondear en bucle, recolectar las tareas pertinentes antes de responder y matar el trabajo irrelevante. El esquema del subagente solo añade que el modo segundo plano devuelve un id de tarea y que `job_output` recolecta el resultado. La autorización y la limpieza del dueño aplican la frontera del runtime con independencia del cumplimiento del prompt.

## Alternativas consideradas

### Herramientas de espera, salida y parada específicas del subagente

Herramientas específicas de la capacidad duplicarían el protocolo de tareas, enseñarían otro hábito de recolectar-y-parar y complicarían las instancias múltiples de provider. El runtime genérico provee el comportamiento requerido sin cambiar la forma de un-provider-por-instancia de la herramienta.

### Supervivencia tras el cierre del dueño

La supervivencia exige estado de tarea persistente, recuperación de la sesión hija, un canal de entrega de resultados tardíos y política para dueños abandonados. La limpieza con ámbito de dueño da al trabajo local al proceso un ciclo de vida claro. Las tareas durables requieren un diseño aparte.

### Sin comprobaciones de dueño para clientes aislados

Los agents y los registros pueden tener ámbito de sesión, pero el registro de tareas y los ids predecibles son globales al runtime. La barrera genérica de dueño se aplica por tanto a los subagentes como a cualquier otro productor.

### Salida incremental del transcript hijo

Transmitir el historial hijo hacia el padre difuminaría la frontera de registro y haría diverger el comportamiento del provider. Esta herramienta expone solo la salida final; la observación más rica pertenece a las herramientas de sesión o de UI.

## Pruebas

La cobertura unitaria fija el mapeo de motivos de parada, el comportamiento de disponer-antes-de-reportar, los fallos de arranque y de resultado, el rechazo de pre-aborto, el desprendimiento de la señal de la llamada iniciadora, la cancelación antes y después de la publicación del provider, la recolección a través de las herramientas reales de tareas, la barrera de preflight sin controlador, el fallo por runtime ausente y el esquema por instancia. La cobertura de instantáneas fija los esquemas visibles para el modelo.

## Consecuencias

El padre puede abanicar delegaciones lentas y recolectarlas a través de los mismos controles de tareas que usa bash. El trabajo del hijo ya no ocupa la llamada de herramienta que lo inició, pero puede consumir recursos hasta ser recolectado, matado o dispuesto por el dueño. La guía del prompt anima a recolectar; la limpieza del dueño provee la frontera dura de ciclo de vida. Los despliegues que exigen delegación síncrona pueden deshabilitar el modo segundo plano por instancia de herramienta.
