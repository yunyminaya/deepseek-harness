# Modo plan

[English](plan.md) | Español

El modo plan es un estado de colaboración por agent (agente) registrado en el log, propiedad de [dsh-plan-mode](../../packages/plan/plan-mode) (`ctx.planMode`, `PlanModeController`): mientras está activo, cada solicitud al modelo incluye una sección de guía propiedad del despliegue. El modo plan es **orientativo**. El [modo sandbox](sandbox.es.md) y la [política de aprobación](approval.es.md) imponen restricciones de forma independiente; ninguno lee ni escribe el estado del plan, así que los despliegues los configuran por separado. El paquete es opcional y el agent loop no depende de él. Aporta la sección de prompt `plan:policy` y registra la herramienta `exit_plan_mode` y el comando `/plan`. La [nota de diseño](../../.agents/notes/implemented/simplification/2026-07-22-plan-specific-collaboration-state.es.md) es la dueña del razonamiento; el [README del paquete](../../packages/plan/plan-mode/README.es.md) es el dueño del detalle de la experiencia de modelo y de las limitaciones.

Source: [`packages/plan/plan-mode/src/index.ts`](../../packages/plan/plan-mode/src/index.ts)

## Estado registrado en el log y recuperación

`plan/mode` (`{ active: boolean }`) es un [evento de sesión](session.es.md) solo de log y de reemplazo completo de valor: duradero y reproducible, nunca presente en el transcript del modelo. `foldPlanMode(events, end?)` devuelve el último valor registrado en el prefijo, o `false` cuando no hay ninguno — el estado en vigor es siempre un fold puro del log de sesión, así que la reanudación, la bifurcación y la compactación lo recuperan sin ningún espejo en vivo, y las UIs observan los flips confirmados a través de `session/event`. La declaración completa del evento está en el [catálogo de eventos de log de persistencia](../persistence-catalog.es.md).

## Selecciones pendientes y el añadido del pre-paso

Como cada evento de sesión va encerrado en un turno, una selección del usuario permanece pendiente hasta que el siguiente pre-paso dentro del turno aceptado la añade antes de la derivación de la solicitud, en el turno que sea. Una selección nunca fuerza la continuación, así que una hecha después del último pre-paso aceptado de un turno se añade en un turno posterior. `set(agent, active)` registra la selección pendiente (un no-op cuando el objetivo es igual al estado registrado o ya pendiente), y `get(agent)` devuelve `{ active: boolean; pending?: boolean }`: el estado registrado usado para componer el paso actual más el estado seleccionado que espera a añadirse.

El único punto de añadido mientras un agent está en ejecución es un listener `agent/pre-step` antepuesto. Observa cada paso de solicitud propuesto, incluidos el paso 1 del turno 1 y los reintentos de recuperación de solicitud, llama primero a los listeners posteriores y solo añade después de que estos acepten el paso. La admisión del prompt ocurre antes de un turno y no puede añadir `plan/mode`, así que una selección hecha en el prompt la añade el primer pre-paso dentro del turno aceptado del turno que inicia. Un fallo de añadido no puede bloquear el turno, y la selección permanece pendiente para un pre-paso dentro del turno aceptado posterior. Una selección de usuario añadida también registra un aviso `user/message` originado en un plugin, pero solo cuando la última cabecera de solicitud registrada describía el otro estado, para que el modelo se entere exactamente de cuándo cambió su contexto y nunca de forma redundante. Una selección hecha después del último pre-paso aceptado de un turno permanece local al proceso y se pierde si el proceso sale antes de otro pre-paso dentro del turno aceptado ([limitación del README](../../packages/plan/plan-mode/README.es.md#known-limitations-and-deferred-work)).

## Configuración

```ts type-equiv
/** Deployment-owned plan guidance. */
interface PlanModeConfig {
  /** Guidance rendered as the `plan:policy` prompt section while plan mode is active. */
  section: string
}
```

Una `section` ausente, vacía o que no sea una cadena, y cualquier clave desconocida, hacen fallar la carga del plugin en lugar de ser ignoradas. Mientras el modo plan está activo, el texto exacto de `section` se renderiza como la [sección de system-prompt](system-prompt.es.md) `plan:policy` en el orden 50; el modo plan inactivo no aporta ningún texto.

## La herramienta de salida y el comando `/plan`

[`exit_plan_mode`](../tool-catalog.es.md#deepseek-aidsh-plan-mode) permanece registrada mientras el modo plan está inactivo, así que entrar o salir del modo plan cambia solo la sección de prompt, nunca el catálogo de herramientas de la solicitud; ejecutarla fuera del modo plan falla. En el modo plan exige un plan markdown completo que empiece por un encabezado `#` y lo presenta para revisión a través del [seam de preguntas al usuario](user-questions.es.md). La aprobación devuelve `{ approved: true }` y registra una salida pendiente silenciosa (sin narración) que se añade en el siguiente pre-paso dentro del turno aceptado. Por tanto, la guía del plan permanece activa durante el resto del lote de herramientas actual del asistente, y el propio resultado de la herramienta informa de la transición. Seguir planificando es una llamada fallida que lleva los comentarios del usuario, así que el modelo revisa y vuelve a presentar; un canal de interacción ausente y una recarga del servicio durante la revisión también hacen fallar la llamada en lugar de salir del modo plan en silencio.

Cuando se compone [`ctx.commands`](commands.es.md), el plugin registra `/plan [off|message]`: `/plan` a secas selecciona el modo plan, cualquier otro mensaje no vacío lo selecciona y luego envía el texto a través de `agent.steer()` para que se convierta en el mensaje de usuario ordinario registrado del siguiente paso bajo la guía del plan, y el argumento exacto `off` selecciona el estado inactivo, lo que también cancela una entrada pendiente antes de que se añada y se vuelva visible para una solicitud.

## El servicio

`ctx.planMode` es el dueño del estado de plan registrado, aplica y narra el estado seleccionado al inicio del paso, y es el dueño de la sección `plan:policy`, del comando `/plan` y de la herramienta de salida estable; las firmas de `get`/`set` están en el [catálogo de servicios](#ctxplanmode--planmodecontroller) generado.

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## API de Cordis

Generado desde la fuente por `scripts/gen-cordis-catalog.ts` (verificado como fresco por `pnpm run verify-cordis-catalog` en doc-sync; regenéralo con `pnpm run gen-cordis-catalog`) — los lados de idioma solo difieren en las rutas de los documentos emparejados específicas de cada localización. Los bloques de firmas usan un recinto `ts cordis-catalog` y conservan el JSDoc original de la fuente; los modos de despacho están definidos en el [primer](../cordis-primer.es.md#dispatch-modes), y la API `ctx` heredada del framework vive en [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxplanmode--planmodecontroller"></a>

### `ctx.planMode` — `PlanModeController`

`ctx.planMode`: es el dueño del estado de plan registrado, aplica y narra el estado seleccionado al inicio del paso, y es el dueño de la sección `plan:policy`, del comando `/plan` y de la herramienta de salida estable. Las UIs observan los flips confirmados a través de `session/event`; no hay ningún espejo en vivo.

```ts cordis-catalog
/**
 * Read the logged plan state and any selected state awaiting the next
 * accepted in-turn pre-step.
 *
 * @param agent The agent to read.
 * @returns Current logged state plus a pending selection, when present.
 */
get(agent: Agent): { active: boolean; pending?: boolean }

/**
 * Select whether plan mode should be active. Between turns the method
 * appends the change immediately because no in-turn pre-step will run until
 * another prompt starts a turn. The open-turn fold is the idle signal:
 * agent status stays `running` through post-turn checkpointing, when no
 * further in-turn pre-step runs. During an open turn the selection remains
 * pending until the next accepted in-turn pre-step. Repeated selection of
 * the current or already-pending state is a no-op.
 *
 * @param agent The agent to switch.
 * @param active Whether plan mode should be active.
 * @returns what happened: `committed` (logged now), `queued` (awaiting the
 * next accepted in-turn pre-step), `cancelled` (an opposite pending selection
 * was cleared; the logged state already matches), or `noop` (already in that
 * state).
 */
set(agent: Agent, active: boolean): 'committed' | 'queued' | 'cancelled' | 'noop'
```

Types: [Agent](core.es.md)

Source: [`packages/plan/plan-mode/src/index.ts`](../../packages/plan/plan-mode/src/index.ts)
<!-- END GENERATED cordis-surface -->
