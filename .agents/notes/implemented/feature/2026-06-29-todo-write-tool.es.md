# Agent Note: La herramienta `todo_write` — lista de tareas del modelo como estado de sesión event-sourced

Status: implemented

[English](2026-06-29-todo-write-tool.md) | Español

## Problema

El harness le da al modelo herramientas de bash y de subagente pero ninguna forma de registrar una lista de tareas estructurada. Una lista de tareas pendientes sirve a dos propósitos de igual jerarquía: dirige al modelo a planificar trabajo de varios pasos y mantener inequívoco el trabajo activo, y le da a un host interactivo una lista de verificación de progreso en vivo. Todos los coding agents de referencia encuestados (claude-code, opencode, codex, oh-my-pi, pi) entregan alguna forma de esto; el harness no tenía nada.

## Decisión

Añadir una herramienta `todo_write(todos: [{ content, status }])` orientada al modelo cuyo estado de lista completa vive en el registro de sesión event-sourced como una nueva variante `todo/write` de `SessionEventMap`. Los hosts interactivos renderizan desde el evento duradero: la TUI lo pliega directamente, el client web lo proyecta en `ConversationSnapshot.todos` ([visualización de todos web](2026-07-23-web-todo-display.es.md)), mientras que el [bridge ACP solo automatización](../simplification/2026-07-23-acp-automation-only-protocol.es.md) omite deliberadamente la presentación de todos.

### Reemplazo de lista completa, estado de tres valores

El modelo envía la lista entera en cada llamada; la lista nueva sustituye a la vieja (last-write-wins en replay). Esta es la forma que usan claude-code V1, opencode y `update_plan` de codex, y la forma en la que el modelo está más entrenado — sin ids por elemento, sin protocolo de deltas. `status` es exactamente `pending | in_progress | completed`, el mismo triple que `update_plan` de codex; también coincidió 1:1 con el `PlanEntryStatus` de ACP mientras el bridge proyectó las listas de tareas como actualizaciones de `plan`, un mapeo retirado con el [contrato ACP solo automatización](../simplification/2026-07-23-acp-automation-only-protocol.es.md).

### El estado en el registro de sesión, no en un servicio

La lista se añade como un evento `todo/write` que transporta la instantánea completa `{ todos }`. El harness es event-sourced — el historial del LLM, las llamadas de herramienta y la estructura de turnos viven en el registro — así que la lista de tareas vive también ahí. Esto compra durabilidad, replay y reconstrucción al reanudar gratis: una sesión reabierta re-deriva el plan vigente desde el último `todo/write` que no está seguido de un `turn/start` posterior ([duración del despoje del plan](2026-07-28-todo-plan-clears-on-next-turn.es.md)), sin backend de persistencia aparte, sin servicio en memoria que rehidratar ni cableado extra. Un servicio `ctx.todos` en memoria tendría que reinventar todo eso. (Los consumidores del registro completo obtienen esta reconstrucción directamente; la ventana paginada del client web la obtiene desde la proyección computada por el host de la página de historial de la cola — véase la [Agent Note de visualización de todos web](2026-07-23-web-todo-display.es.md).)

### NO es un evento de superficie

`todo/write` queda deliberadamente excluido de `SurfaceEventType`. La superficie es la proyección que produce el historial de mensajes del LLM (`deriveMessages()`); una escritura de todos no produce ningún mensaje de conversación. Así que no lleva `surfaceOp`, jamás se une a la superficie ordenada y jamás alcanza `deriveMessages()` — es estado *de UI* duradero y reproducible que viaja junto a la conversación sin formar parte de ella. (Las invariantes de modo desarrollo siguen exigiendo que repose dentro de un turno abierto, cosa que siempre hace: se añade a mitad de paso durante una llamada de herramienta.)

### Descartado frente a claude-code V1: `activeForm`, id, prioridad

El elemento de claude-code V1 es `{ content, status, activeForm }`; más tarde (V2) creció ids, dependencias y titularidad — pero solo para soportar *swarms* de agents (con respaldo en disco, protegidos con locks, mutación por elemento). Esta herramienta mantiene el elemento al mínimo: `{ content, status }`. Sin `activeForm` (la etiqueta de presente continuo) — la UI muestra `content`; sin id — el reemplazo de lista completa no necesita identidad estable; sin prioridad — esa solo fue jamás un requisito de cable del `PlanEntry` de ACP, sintetizada como constante en la frontera del bridge en lugar de modelada, y se fue con esa proyección. Cada campo descartado es una cosa menos que el modelo debe producir en cada llamada.

### Un solo dueño — sin maquinaria de swarm (YAGNI)

Cada lista pertenece a la sesión del agent que llama, y las llamadas que no son de agent se rechazan. No hay ámbito compartido, resolver ni protocolo de deltas. Las listas entre agents exigirían deltas de registro por elemento y selección explícita de ámbito, así que siguen siendo un diseño futuro aparte.

### Validación: el punto medio barato

El schema exige tipo/requerido/enum. Más allá de eso, `execute` rechaza `content` vacío o duplicado y, cuando `allowParallelInProgress` es `false`, más de una tarea activa. La ordenación y mantener la lista al día siguen siendo disciplinas del modelo expresadas en la descripción de la herramienta. Una escritura rechazada devuelve un resultado `isError` para que el modelo se autocorrija. La política de despliegue requerida y la independencia de la invariante duradera respecto a ella las posee la [Agent Note de tareas paralelas en curso](2026-07-26-todo-parallel-in-progress.es.md).

## Por qué no hay entrada en el catálogo cordis ni `@mode`

`todo/write` es un miembro de `SessionEventMap`, no un evento de primera clase `interface Events` de cordis. El generador del catálogo (`scripts/gen-cordis-catalog.ts`) escanea declaraciones `interface Events`; una variante de `SessionEventMap` viaja en el emit existente `session/event` y no produce ninguna fila nueva de catálogo. Así que no lleva etiqueta `@mode` (que el generador exige solo a miembros de `interface Events`) — añadir una no significaría nada.

## Pruebas

Cuatro niveles:
- **Unit** — el evento de sesión (append/clonación de instantánea/last-write-wins/no en superficie); la herramienta (forma del schema, validación de argumentos vía el `ctx.tools.execute` real, validación de valores, el append del evento + reemplazo, rechazo sin agent, `presentCall`, seguridad ante HMR); y el plegado de la TUI.
- **Camino Real-Loader** — el plugin corrido a través de `Loader.unwrapExports`, afirmando que la forma de la exportación del namespace sobrevive (TIENE `inject`, así que un default perdido tumbaría la carga — postmortem/0001).
- **Integración de loop completo** — un modelo mock con guion llama `todo_write` a través del agent loop real; el evento `todo/write` aterriza y una segunda llamada lo sustituye.
- **Reanudación/replay** — un `todo/write` persistido se pliega de vuelta en la lista de tareas actual.
- **e2e con clave + snapshots** — un prompt real induce `todo_write`; los snapshots ensamblados fijan el evento del registro y la renderización interactiva.

## Alternativas consideradas

- **Servicio `ctx.todos` en memoria** — reinventaría la durabilidad, el replay y la reconstrucción al reanudar que el registro da gratis.
- **Protocolo de deltas por elemento** — solo necesario para una lista compartida de múltiples dueños, que está fuera de ámbito; el reemplazo de lista completa es más simple y coincide con las referencias.
- **Herramienta en `core/`** — `todo_write` es una herramienta de extensión que se registra en `ctx.tools`, no parte de la spine; vive en su propio grupo `packages/todo/` como otras familias de herramientas.

## Consecuencias

La lista de tareas es estado de sesión duradero y reproducible: un host interactivo la re-deriva desde el último `todo/write` persistido, y el registro — no la memoria del plugin — es la fuente única de verdad. El reemplazo de lista completa significa una llamada de herramienta por actualización con last-write-wins; no hay ningún protocolo de deltas que reconciliar. El evento permanece fuera de la superficie del modelo, así que una actualización de todos jamás perturba el historial derivado del modelo — el modelo ve solo su propia llamada y resultado de herramienta.
