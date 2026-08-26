# Agent Note: El stop Web conserva la Queue pendiente

Status: implemented

English | [中文](2026-07-31-web-stop-preserves-queue.zh.md)

## Problema

El botón de stop Web alcanzaba `session.cancel`, que mapeaba al amplio `agent.cancel({ kind: 'user' })`. Durante un turno activo, los envíos ordinarios del composer ya se aceptan como ocurrencias de Queue direccionables de forma independiente. La cancelación amplia descartaba cada ocurrencia cuando el usuario pretendía detener solo la generación actual, conflando la interrupción del turno con la operación explícita de borrado de la Queue.

El browser no puede reparar esa pérdida re-enviando filas visibles. No posee su `InboxItemId` vivo, su política de wake ni su carrera de claim, y un re-envío puede duplicar trabajo que el Host ya reclamó.

## Decisión

`session.cancel` es el stop de turno-activo de la API Web del Host para sesiones ordinarias. Rechaza subagentes respaldados por sesión con `agent-busy`; si no, llama `agent.cancel({ kind: 'user' }, { keepInbox: true })`, preservando el trabajo pendiente del inbox mientras aborta cooperativamente el turno actual. La opción subyacente preserva las entradas en cola y de steering; la proyección Web de Queue sigue exponiendo solo entradas en cola.

El AgentLoop no arranca ningún turno de reemplazo concurrente. Cierra y flushea el turno interrumpido, alcanza la quietud de cancelación, y luego reclama la siguiente ocurrencia en cola despertadora a través de su driver FIFO existente. Ese claim emite `agent/inbox/dequeue`, así que la instantánea autoritativa `session/queue` del Host retira la fila reclamada y deja visible la cola restante. El browser ni re-envía ni promociona ninguna fila. El trabajo que ignora la cancelación retrasa este relevo hasta asentarse.

Este mapeo cambia solo el endpoint `session.cancel` del Host usado por clientes Web. El contrato por defecto `Agent.cancel()` sigue siendo amplio, ACP y TUI conservan sus políticas de cancelación existentes, y `AgentHandle.dispose()` sigue limpiando el trabajo pendiente durante el teardown. La eliminación de filas de Queue sigue siendo la acción Web explícita para descartar una ocurrencia pendiente.

## Alternativas consideradas

**Conservar la cancelación amplia para el botón de stop.** Descartado porque detener una generación no debe destruir intención de usuario encolada de forma independiente; la Queue ya posee el borrado explícito.

**Re-enviar la siguiente fila desde el browser tras la cancelación.** Descartado porque el Host posee la identidad de ocurrencia y el orden de claim. La resubmisión del cliente puede duplicar trabajo, reordenar el FIFO, o correr una carrera contra un dequeue autoritativo.

**Arrancar el siguiente turno antes de que el trabajo cancelado alcance quietud.** Descartado porque dos turnos mutarían concurrentemente un log de sesión y compartirían recursos propiedad del Agent. La cancelación cooperativa espera con verdad a que el trabajo activo se asiente.

**Añadir una opción wire de cancelación amplia vs preservadora.** Descartado hasta que el producto Web tenga una interacción separada de «detener y limpiar Queue». El botón de stop existente tiene una política, mientras el borrado por fila ya suministra el control de descarte actual.

## Verificación

La cobertura del AgentLoop sostiene un stream de modelo activo, encola dos turnos despertadores, cancela con `keepInbox`, y fija las razones de turno abortado-luego-completado, el orden FIFO de user-messages, la ausencia de eventos de descarte, y el estado eventual idle. El escenario keyless Web conduce la composición construida sobre HTTP/SSE: detiene un turno colgado, observa arrancar la siguiente ocurrencia en cola mientras la cola permanece visible, detiene ese turno, y observa completar la ocurrencia final en cola. Su instantánea de accesibilidad fija el estado intermedio de Queue-preservada.

## Consecuencias

El stop Web conserva la intención encolada aceptada y la avanza automáticamente tras el asentamiento veraz de la cancelación. Las filas de Queue pueden permanecer visibles mientras el trabajo activo no-cooperativo se agota, y el steering externo preservado por la misma opción de inbox puede entrar al siguiente turno admitido aunque Web no renderice steering en QueueDock. Una futura interacción de limpieza masiva exige una acción explícita de producto en vez de sobrecargar el stop.
