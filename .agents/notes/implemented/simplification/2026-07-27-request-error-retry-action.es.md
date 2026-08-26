# Agent Note: Acción de reintento ante errores de solicitud

Status: implemented

[English](2026-07-27-request-error-retry-action.md) | [中文](2026-07-27-request-error-retry-action.zh.md) | Español

## Problema

La recuperación de solicitudes al modelo se decidía dentro de `agent/request-error` pero se comunicaba a través de `Agent.retry()`. Ese comando público era válido durante una ventana estrecha del waterfall y en reposo, rechazaba otros estados en ejecución, y exigía que `ReactLoopAgent` retuviera una ventana de reintento mutable junto al resultado del waterfall. Los plugins de recuperación eran los únicos llamadores de producción, así que la capacidad más amplia del agente vivo exponía estados y comportamiento no relacionados con su decisión de política.

## Decisión

`agent/request-error` devuelve `RequestErrorAction`, cuya acción de manejo es `{ kind: 'retry' }`; el `undefined` por defecto mantiene terminal el turno fallido. Un listener que no es dueño del fallo llama a `next()`. Un listener que lo es dueño realiza cualquier reparación esperada y devuelve la acción de reintento sin delegar.

El loop lee la acción después de que el waterfall se asienta, cierra el turno fallido, y abre un turno de reintento desde el historial durable. Vuelve a comprobar la señal de turno al consumir la acción, así que la cancelación o disposición durante la recuperación impide el reintento aunque un listener lo devuelva después. Una recuperación que lanza nunca produce una acción.

`Agent` y `ReactLoopAgent` no exponen ningún método `retry()`. El trabajo nuevo ordinario entra por `followup()`, `steer()` e `inject()`; solo un fallo manejado de solicitud al modelo puede abrir un turno de reintento sin prompt.

## Alternativas consideradas

**Conservar `Agent.retry()` como comando de recuperación.** Los guards de runtime pueden restringir el comando a la ventana de request-error, pero la interfaz sigue anunciando una operación de reinvocación en reposo sin consumidor de producción y el loop sigue necesitando estado mutable de canal lateral para recuperar una decisión ya propiedad del waterfall.

**Devolver una acción terminal explícita.** `undefined` ya representa el default no manejado del waterfall y compone directamente a través de `next()`. Un segundo valor `{ kind: 'fail' }` no añadiría comportamiento ni información de propiedad distintos.

## Consecuencias

La propiedad de la recuperación, la reparación asíncrona y la decisión de reintento comparten una única ruta de retorno tipada. La interfaz del agente vivo y el loop concreto pierden la capacidad de reinvocación en reposo y el estado de ventana de reintento. Los llamadores no pueden reiniciar trabajo fallido arbitrario no relacionado con solicitudes sin enviar un prompt posterior, mientras que las políticas transitorias y de desbordamiento de contexto conservan turnos de reintento numerados, reconstrucción desde historial durable, presupuestos privados finitos y precedencia de cancelación.

Pruebas enfocadas del agent-loop fijan el encadenamiento de reintentos, la caída terminal, el fallo de recuperación y las carreras de cancelación. Las suites llm-retry y compaction-basic fijan sus devoluciones de acción propiedad de la política, y las integraciones ACP, goal-round-driver y plan-mode fijan la adopción del turno sucesor.
