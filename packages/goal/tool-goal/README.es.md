# @deepseek-ai/dsh-tool-goal

[English](README.md) | Español

Las herramientas de control orientadas al modelo para [`ctx.goals`](../goal/README.es.md): `get_goal`, `create_goal` y `update_goal`. La [Agent Note de las herramientas de objetivo](../../../.agents/notes/implemented/feature/2026-07-19-model-facing-goal-tools.es.md) es la dueña de la división de autoridad y de la UX con forma de Codex.

## Herramientas

- `get_goal()` devuelve el objetivo actual o `null`, incluidos el id/revisión de comparar-y-establecer, la fase duradera, las rondas de objetivo admitidas/limitadas, cualquier motivo de bloqueador y la activación local del proceso actual.
- `create_goal(objective, max_goal_rounds?)` crea un objetivo a partir de un turno humano directo de nivel superior. El modelo puede inferir la intención de objetivo de larga duración sin una frase de comando exacta; los turnos no humanos y los subagentes se rechazan en la ejecución.
- `update_goal(goal_id, revision, action, objective?, max_goal_rounds?, blocked_reason?)` admite `edit`, `pause`, `resume`, `complete` y `blocked`. Los reemplazos pertenecen solo a `edit`; `blocked_reason` solo es obligatorio para `blocked` y se persiste con el código estable `model-reported`. En el schema estricto, los rellenos de cadena vacía y de cero cuentan como omitidos, mientras que los valores significativos siguen limitados a su acción.

Todas las llamadas son exclusivas, de modo que un lote ordenado por el modelo observa las mutaciones anteriores y sus revisiones nuevas. Los clientes de UI reciben tarjetas genéricas puras: read para `get_goal`, other para las mutaciones. Las tarjetas de mutación seleccionan el primer valor de acción significativo y, en caso contrario, muestran el id de objetivo, así que los rellenos aceptados nunca producen entrada en blanco.

Los tres valores canónicos coinciden con el JSON compacto ya renderizado para los llamantes Native: `{ goal: null }` o `{ goal: { id, revision, objective, phase, roundsStarted, maxGoalRounds, blockedReason? }, activation }`. Por tanto, los consumidores programáticos reciben la misma estructura de dominio sin analizar el JSON renderizado.

Una ronda de objetivo autónoma que informa con éxito `complete` o `blocked` marca esa ejecución de herramienta con `concludeTurn()` para que el turno físico se detenga después del paso. Las mutaciones humanas directas nunca contribuyen a esta detención: el asistente puede reconocer el cambio y el steering humano concurrente sigue disponible para el loop.

## Autoridad

La ejecución exige el `exec.agent` en vivo exacto, su iniciador `AgentRegistry` heredado, el estado de ejecución y un turno abierto. Create, edit, pause y resume exigen además un mensaje `{ kind: 'user' }` aceptado o un evento de steering en el turno actual de un agent raíz del runtime. El linaje de fork duradero no degrada una raíz reanudada; la propiedad de subagente en vivo sí lo hace.

`{ kind: 'user' }` es una atestación del host. `Agent.followup()` y `steer()` la asignan cuando su llamante omite una fuente, así que los plugins, los planificadores y otros productores no humanos deben pasar su propia fuente en lugar de heredar la autoridad humana.

Complete y blocked también aceptan la ronda de objetivo actual exacta: un `user/message` originado por el objetivo cuyo id, revisión y ronda coinciden con el objetivo actual plegado. Una llamada blocked de ronda de objetivo se rechaza mecánicamente hasta `blockedAfterConsecutiveRounds`; el modelo juzga si la misma condición persistió de hecho y debe describirla en `blocked_reason`. La autoridad humana directa puede detener un objetivo de inmediato.

## Configuración

```yaml
- id: tool-goal
  name: '@deepseek-ai/dsh-tool-goal'
  config:
    blockedAfterConsecutiveRounds: 3
```

El valor debe ser un entero seguro positivo. Aporta tanto el límite inferior duro del autobloqueo del modelo como el número nombrado en la orientación al modelo.

## Experiencia de modelo

### Prompt del sistema

#### Lo que ve el modelo

Una política de objetivo fija dice cuándo la intención humana semántica justifica la creación, exige refs exactas de leer-antes-de-actualizar, explica el rearme después del resume/fork y limita las afirmaciones de finalización/bloqueo. El umbral configurado se interpola en esa orientación.

##### Política de objetivo

```markdown
Use goal tools for one long-running completion objective in the current session. create_goal may infer goal intent from a direct human request in any language; do not create a goal for routine single-turn work. Call get_goal before update_goal and copy its exact goal_id and revision. After session resume or fork, an active goal is disarmed: when a human asks to continue or resume in any wording or language, use update_goal action resume to rearm it. Mark complete only when the objective is actually achieved. Mark blocked only after the same blocking condition persists for at least 3 consecutive goal rounds, and report that concrete condition in blocked_reason; difficulty, uncertainty, or useful remaining work is not blocked.
```

#### Efecto en tokens

Pequeño coste fijo de entrada en cada petición donde el registro de prompt de este plugin está en ámbito.

#### Efecto en la KV cache

Estable en el prefijo mientras el ámbito del plugin, el umbral configurado y el texto de orientación no cambian. La activación, el desmontaje o los cambios de configuración pueden invalidar la reutilización de esta sección de prompt.

### Schemas y resultados de herramienta

#### Lo que ve el modelo

Los [schemas generados de `get_goal`, `create_goal` y `update_goal`](../../../docs/tool-catalog.es.md#deepseek-aidsh-tool-goal). Los resultados con éxito son JSON compacto. Una mutación añade el evento duradero `goal/change` del dominio de objetivo sin poner contexto de modelo en cola. `activation` en un resultado es una observación en vivo y nunca se convierte en autoridad de reproducción.

#### Efecto en tokens

Coste fijo de schema más un resultado compacto por llamada. La mutación duradera no añade contexto visible para el modelo independiente.

#### Efecto en la KV cache

Los schemas son estables en el prefijo mientras sus definiciones y visibilidad no cambian. Las llamadas y los resultados se añaden después del prefijo de petición reutilizable sin invalidar las entradas anteriores.

## Limitaciones conocidas y trabajo pendiente

- **La intención semántica sigue siendo juicio del modelo** — la ejecución puede demostrar que el turno actual contiene un mensaje humano directo, no si la petición es lo bastante sustancial como para merecer un objetivo.
- **El bloqueo por la misma condición sigue siendo juicio del modelo** — el runtime aplica el recuento distinto de rondas admitidas, no la equivalencia semántica de los obstáculos; un evaluador independiente queda pendiente.
- **Sin planificación ni renderizado humano directo** — estas herramientas solo mutan estado; el driver de misma sesión y [`dsh-command-goal`](../command-goal/README.es.md) son consumidores independientes del mismo dominio.
- **La autoridad de ronda de objetivo exige un driver** — la ruta autónoma de `complete`/`blocked` está inactiva salvo que un driver de continuación admita turnos de usuario originados por el objetivo; montar solo este paquete de herramientas no los crea.
- **El registro de prompt es independiente del filtrado** — un ámbito puede ocultar las herramientas conservando su orientación salvo que el despliegue dé ámbito a ambos registros juntos.
