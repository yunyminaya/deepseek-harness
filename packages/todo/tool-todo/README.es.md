# @deepseek-ai/dsh-tool-todo

[English](README.md) | Español

La herramienta `todo_write` orientada al modelo: la lista completa de tareas del agent, reemplazada por entero en cada llamada.

## Qué hace

Registra una herramienta, `todo_write(todos: [{ content, status }])`, en `ctx.tools`. El modelo envía la lista ENTERA en cada llamada — no hay actualizaciones parciales ni ediciones por elemento. Cada llamada añade un evento `todo/write` (la instantánea completa de la lista) al log de sesión del agent que llama mediante `agent.session.append('todo/write', { todos })`; la lista actual es el evento más reciente de ese tipo (la última escritura gana en el replay).

`status` es uno de `pending`, `in_progress` o `completed`.

## Dueño único

La lista pertenece a la ÚNICA sesión de agent que llamó a la herramienta. No hay ámbito de subagente/compartido/swarm: un llamador que no es agent (sin `exec.agent`) no tiene dónde escribir la lista y es rechazado. Es un límite de ámbito deliberado — ver la Agent Note.

## Configuración

`allowParallelInProgress` es obligatorio: cada composición debe elegir si varios todos pueden estar `in_progress` a la vez. Es una elección de despliegue, no una regla fija: si las tareas activas concurrentes son legítimas depende de la concurrencia de runtime que la herramienta no puede observar. Usa `true` para agents que pueden desplegar trabajo en abanico y `false` para imponer la disciplina de un solo activo.

La bandera mueve juntas la instrucción orientada al modelo y la entrada aceptada — `true` pide al modelo que marque cada tarea trabajada activamente y acepta cualquier número, `false` pide exactamente una y rechaza una llamada que marque más con `Error: invalid todos: at most one task may be in_progress (got <n>)`. El invariante del log durable NO la sigue: un log escrito mientras el trabajo paralelo estaba permitido debe seguir reproduciéndose después de que un despliegue endurezca la política, así que el invariante permanece mudo sobre el número de activos.

## Validación

Más allá de las comprobaciones de tipo/obligatoriedad/enum del schema, `execute` rechaza un `content` vacío o duplicado, y cualquier clave de elemento más allá de `content`/`status` — una forma de elemento extendida (ids, anidamiento) falla de forma ruidosa en lugar de aplanarse en silencio, manteniendo la instantánea registrada igual a lo que el modelo cree que escribió. Cuántas tareas pueden estar `in_progress` a la vez es decisión del despliegue (§ Configuración): una composición que elige `true` permite que el trabajo paralelo (subagentes concurrentes, comandos en segundo plano) marque varias tareas simultáneamente. El orden y la disciplina de mantener la lista al día quedan al modelo a través de la descripción de la herramienta.

## Renderizado

El resultado canónico es `{ todos, counts: { pending, inProgress, completed } }`; su renderizador nativo devuelve el acuse de actualización compacto. La herramienta también escribe el evento de sesión completo `todo/write`. Las UIs se suscriben al stream de eventos y renderizan esa lista durable por sí mismas: el [cliente web](../../client/ui-conversation) muestra una franja de plan más una fila de herramienta dedicada fuera del plan permanente — el último `todo/write` sin un `turn/start` posterior ([visualización](../../../.agents/notes/implemented/feature/2026-07-23-web-todo-display.es.md), [ciclo de vida](../../../.agents/notes/implemented/feature/2026-07-28-todo-plan-clears-on-next-turn.es.md)).

## Proyección de sesión

Cuando la composición monta `ctx.sessionProjections` ([`@deepseek-ai/dsh-session-projection`](../../session/session-projection/README.es.md)), este paquete registra la unidad de proyección `todos` bajo un hijo inyectado: `init` = `null` (sin escritura aún), `apply` = toma la lista entera de cada `todo/write` y la limpia a `null` en cada `turn/start` (plan permanente; `turn/end` conserva la lista de verificación terminada; cualquier otro evento devuelve la misma referencia de estado), `view` = identidad, `stateVersion` = 2. La clave se fusiona en `SessionProjectionMap` aquí (a través del outlet `/types` del paquete de Service Definition); el framework conduce la unidad y los carriers sirven el valor en la página de cola del historial y en el marco de push `session/projection`. Las composiciones sin el registro no se ven afectadas. Justificación del ciclo de vida: [el plan de todos se limpia en el siguiente turno](../../../.agents/notes/implemented/feature/2026-07-28-todo-plan-clears-on-next-turn.es.md).

## Forma de exportación

Un plugin de función/namespace: exporta `name` / `inject` / `apply` y NO default. Un `export default` accidental colapsaría el módulo a través del `unwrapExports` del Loader y perdería `inject` (ver [docs/postmortem/0001](../../../docs/postmortem/0001-acp-default-export-drops-inject.es.md)).

## Model Experience

### Schema de la herramienta

#### Lo que ve el modelo

El modelo ve el [schema generado de `todo_write`](../../../docs/tool-catalog.es.md#deepseek-aidsh-tool-todo).

#### Efecto de tokens

Coste de schema fijo en cada petición donde la herramienta es visible.

#### Efecto de caché KV

Estable respecto al prefijo mientras la definición y la visibilidad no cambian. Las restricciones de ciclo de vida del plugin o de ámbito pueden invalidar la reutilización de este schema.

### Historial de llamadas de herramienta y resultado

#### Lo que ve el modelo

Cada llamada de herramienta del assistant conserva la lista completa de reemplazo en sus argumentos. El éxito devuelve exactamente `Updated todo list: <pending> pending, <inProgress> in progress, <completed> completed.` Los fallos estables son ``Error: invalid todo: `content` must be a non-empty string``, `Error: invalid todos: duplicate content "<content>"`, `Error: todo_write requires an owning agent session` y — solo donde el despliegue fijó `allowParallelInProgress: false` — `Error: invalid todos: at most one task may be in_progress (got <n>)`. El evento de sesión completo `todo/write` es estado de UI y replay, no un segundo mensaje de modelo.

#### Efecto de tokens

El crecimiento de tokens escala con cada lista completa que el modelo envía, y esos argumentos de llamada permanecen hasta la compactación. El resultado en sí es pequeño y de forma fija.

#### Efecto de caché KV

De solo añadido; el contenido recién visible sigue el prefijo de petición reutilizable y no invalida las entradas existentes de la caché KV.

## Limitaciones conocidas y trabajo diferido

- **Solo ámbito de dueño único** — la lista pertenece a la única sesión de agent que llama; los ámbitos de subagente/compartido/swarm son un recorte deliberado (ver § Dueño único), y un llamador que no es agent es rechazado.
- **La forma del elemento es deliberadamente mínima** — `content` más `status` de tres estados; el reemplazo de lista completa no necesita id estable, prioridad ni campos de formulario activo.
- **El reemplazo de lista completa es la única operación** — sin actualizaciones parciales, sin herramienta de lectura; el modelo debe reenviar la lista entera en cada llamada.
