# Flujo de trabajo

[English](workflow.md) | Español

El seam de flujo de trabajo permite a un agent (agente) ejecutar un SCRIPT de orquestación escrito por el modelo que inicia subagentes (subagent). Como [subagente](subagent.es.md), es **una capacidad opcional** que no forma parte del agent loop (bucle del agente), por lo que sus tipos y operaciones viven aquí y no en [core.md](core.es.md). Como bash, admite UNA implementación de motor por contexto para proveer `ctx.workflowEngine`; no existe un registro de providers con nombre (un segundo motor reemplaza al primero mediante la configuración de plugins en lugar de ejecutarse junto a él).

Service Definition: [dsh-workflow](../../packages/workflow/workflow) (`ctx.workflowEngine` + el vocabulario que sigue). El Service Provider es [dsh-workflow-worker-thread](../../packages/workflow/workflow-worker-thread) (un motor de `node:worker_threads` — un hilo de trabajo por ejecución, con el contexto vm del script dentro); el Consumer orientado al modelo es [dsh-tool-workflow](../../packages/workflow/tool-workflow). La propuesta y su fundamento: [la Agent Note de dynamic-workflows](../../.agents/notes/implemented/feature/2026-07-05-dynamic-workflows.es.md).

Fuentes: el vocabulario seguro para navegador en [`packages/workflow/workflow/src/types.ts`](../../packages/workflow/workflow/src/types.ts), y los handles de solicitud de Host y de ejecución activa en [`runtime-types.ts`](../../packages/workflow/workflow/src/runtime-types.ts).

## La solicitud de inicio

Lo que un llamador pide al iniciar una ejecución. La herramienta de flujo de trabajo ordinaria lo construye a partir de la llamada `{ script, meta, args }` del modelo más el agent que la invoca; los consumidores especializados también pueden elegir un `subagentProvider` de ámbito de motor y reducir `maxTotalAgents` para la ejecución, pero el script no puede observar ni reemplazar ninguna de las dos políticas. `meta` y `args` son DATOS JSON planos (el motor valida `meta` contra su schema y rechaza de forma sonora ANTES de que corra nada — nunca se evalúa texto de script para obtenerlos). `parent` es OBLIGATORIO — todos los hijos que el script inicie se le atribuyen a él, y el cwd, el linaje y la profundidad pasan por el [seam de subagente](subagent.es.md).

```ts type-equiv
/**
 * What a caller asks for when starting a workflow run. `meta` and `args` are
 * plain JSON data by the seam contract. `parent` is required because every
 * `agent()` spawned by the script is attributed to that live Agent.
 */
interface WorkflowStartRequest {
  /** The plain-JS script body (top-level await allowed; ends with `return <json-value>`). */
  script: string
  /** The workflow's identity block, as plain JSON data (shape-validated by the engine). */
  meta: WorkflowMeta
  /** Optional input exposed verbatim to the script as the `args` global. */
  args?: unknown
  /** Optional engine-wide child-provider override for this run. */
  subagentProvider?: string
  /** Optional per-run total-child ceiling. */
  maxTotalAgents?: number
  /** The agent on whose behalf the run executes (parent of every child). */
  parent: Agent
  /** Cancels the run when aborted. */
  signal?: AbortSignal
}
```

## La identidad del flujo de trabajo: `WorkflowMeta`

El bloque de identidad que viaja como datos en la solicitud de inicio (el parámetro `meta` de la herramienta; el vocabulario de campos coincide con el bloque meta de dynamic-workflows de Claude Code). `phases` es solo vocabulario de progreso: las llamadas a `phase()` hacen coincidir títulos para los observadores; no implica ninguna estructura de ejecución.

```ts type-equiv
/**
 * The script's identity block, provided as plain JSON data alongside the
 * script body (the model-facing tool carries it as its `meta` parameter) and
 * validated by the engine before the body runs. `name`/`description` are
 * required; the rest is optional annotation. The field vocabulary matches the
 * Claude Code dynamic-workflows meta block.
 */
interface WorkflowMeta {
  /** Short kebab-case workflow name (display + persistence key). */
  name: string
  /** One-line description of what the workflow does. */
  description: string
  /** Optional guidance on when this workflow applies (shown in listings). */
  whenToUse?: string
  /** Optional phase declarations matched by `phase()` calls. */
  phases?: WorkflowPhase[]
}
```

## El resultado final: `WorkflowResult`

El resultado de una ejecución, resuelto por `WorkflowRun.result`. `value` es el valor de retorno materializado del script — datos JSON planos del reino del host (`null` cuando el script no devolvió nada) — con significado solo para `completed`. `stopReason` es una unión CERRADA (propiedad del motor; los consumidores pueden agotarla): `completed` | `cancelled` | `error`. Una razón distinta de `completed` lleva el fallo en `error`, y el consumidor la mapea a un resultado de herramienta `isError` en lugar de informar la salida parcial como éxito.

```ts type-equiv
/**
 * The outcome resolved by a live workflow run. `value` is
 * the script's materialized return value (plain host-realm JSON data; `null`
 * when the script returned `undefined`) — meaningful only for `completed`.
 * A non-`completed` reason carries the failure in `error`; the consumer maps
 * it to an `isError` tool result rather than reporting partial output.
 */
interface WorkflowResult {
  /** The script's return value (host JSON data; `null` for no return). */
  value: unknown
  /** Why the run settled. */
  stopReason: WorkflowStopReason
  /** The failure message (present iff `stopReason` is not `completed`). */
  error?: string
  /**
   * How many `agent()` calls the run accepted over its whole lifetime. On a
   * graceful settlement this is the script-side count (calls still queued for
   * a concurrency slot included); on a termination path (grace force-settle,
   * worker death) it degrades to the host-observed count — calls queued
   * inside a terminated script are unknowable then.
   */
  agentsStarted: number
}
```

## Una ejecución activa: `WorkflowRun`

El handle que retiene el consumidor mientras se ejecuta un script. El consumidor espera `result`, puede llamar a `cancel` a mitad de ejecución y DEBE llamar a `dispose` en todos los caminos. `result` NO rechaza — un fallo del script se resuelve con `stopReason: 'error'` — y una vez cancelada la ejecución, CONCLUYE dentro del período de gracia acotado del motor aunque el propio script nunca concluya (el motor fuerza la conclusión con `cancelled`; el motor de hilos de trabajo termina entonces el hilo del script), así que un consumidor que espera `result` nunca queda atascado después de una cancelación. `dispose()` = cancelación + esa conclusión acotada + la quiescencia de los hijos; nunca se cuelga con un script atascado.

```ts type-equiv
/**
 * Holder-owned live workflow. `result` never rejects; consumers may cancel
 * and must call idempotent `dispose()` to await script and child quiescence.
 */
interface WorkflowRun {
  readonly id: WorkflowRunId
  /** The validated meta block available before the script body runs. */
  readonly meta: WorkflowMeta
  readonly result: Promise<WorkflowResult>
  /** Cancel the run and its children. */
  cancel(reason?: string): void
  /** Cancel if needed and await bounded settlement and cleanup. */
  dispose(): Promise<void>
}
```

## Disciplina de fallos: `WorkflowError.fatal`

El mal uso de un hook dentro de un script — argumentos incorrectos, opciones de `agent()` desconocidas o diferidas, un schema fuera del [subconjunto de salida estructurada](../../packages/core/tools/README.es.md), un tope saltado, un fallo de inicio del seam, una cancelación — lanza un `WorkflowError` con `fatal: true`. Los combinadores `parallel()`/`pipeline()` VUELVEN A LANZAR los errores fatales en lugar de mapear el elemento a `null`: una opción mal escrita debe matar el script de forma sonora, nunca disolverse en algo que se lea como un fallo ordinario de un hijo. El `null` por elemento está reservado para los fallos de ejecución de los hijos (una razón de parada distinta de `completed`) y para los errores ordinarios de script dentro de una etapa.

## Eventos

Los eventos `workflow/*` (`workflow/start`, `workflow/phase`, `workflow/log`, `workflow/agent-start`, `workflow/agent-end`, `workflow/end` — ver el [catálogo de eventos](#cordis-surface)) son **emisiones de solo observación** que transportan instantáneas de DATOS: cada payload comienza con `WorkflowRunInfo` (id + meta), nunca con el `WorkflowRun` activo, de modo que un suscriptor no puede obtener `cancel`/`dispose`, y `workflow/end` omite deliberadamente el valor de resultado (un listener que observa resultados no debe recibir un alias mutable del resultado del llamador). Cada emisión está contenida por listener — un suscriptor que lanza una excepción se registra en el log, nunca se propaga, y no puede dejar sin eventos a los listeners registrados después — y cada listener recibe su propio clon del payload, de modo que mutarlo no corrompe ni al motor ni a los demás listeners; la contención espeja `subagent/start`/`subagent/end`.

## Registros duraderos de Chat

El consumidor de nivel superior `dsh-tool-workflow` proyecta hechos de visualización en la Session principal que lo invoca sin cambiar la titularidad de la ejecución. Escribe `tool-workflow/run-start` después de que una ejecución es aceptada, empareja el inicio y el fin de los miembros por `runId + seq`, y escribe `tool-workflow/run-end` solo después de que el resultado se conoce y la disposición alcanza la quiescencia. Las llamadas de transporte anidadas no escriben ningún registro. El primer fallo de append desactiva las escrituras posteriores para esa ejecución, de modo que el log permanece vacío o como un prefijo continuo legal y el resultado de la herramienta no cambia.

`dsh-tool-workflow/invariant` valida el mismo protocolo antes de la confirmación (commit) en vivo y cuando se carga una Session: un inicio por ejecución, secuencias de miembros positivas y únicas, finales de miembros emparejados, ninguna ejecución que termine con miembros abiertos y ninguna actualización después del fin de la ejecución. Un final de miembro o de ejecución ausente en la cola del log es evidencia válida de interrupción, no corrupción.

`dsh-client-ui-workflow-run` pliega los cuatro eventos a través del motor de Conversation Node en un único nodo de Chat `workflow-run` anclado en la secuencia de run-start, después del nodo original de la herramienta de flujo de trabajo. Los grupos de fases provienen solo de inicios de miembros reales y conservan las cadenas exactas, incluida la distinción entre una fase omitida y `''`. Las Locations cerradas convierten los hechos finales ausentes en una presentación de interrumpido. El [README del paquete UI](../../packages/client/ui-workflow-run/README.es.md) es dueño del comportamiento de divulgación, de estado y de navegación local dentro del mismo padre.

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxworkflowengine--workflowengine-abstract-seam"></a>

### `ctx.workflowEngine` — `WorkflowEngine` (abstract seam)

Workflow Service Definition contract. Invalid requests throw before publication; a live run is holder-owned, its result never rejects, cancellation and disposal are bounded, and disposal waits for child cleanup within that bound. Lifecycle listener failures are contained, and `workflow/end` fires exactly once as the result settles.

```ts cordis-catalog
/**
 * Parse and execute a workflow script.
 * @param request - the script, its `args`, the parent agent, and an
 *   optional cancel signal.
 * @returns the live run; its `result` resolves when the script settles.
 */
abstract start(request: WorkflowStartRequest): WorkflowRun
```

Source: [`packages/workflow/workflow/src/index.ts`](../../packages/workflow/workflow/src/index.ts)

<a id="workflow-events"></a>

### `workflow/*` events

<a id="workflowagent-end--emit"></a>

#### `workflow/agent-end` — emit

One `agent()` call settled (clean result, child failure, or run cancellation). Paired with Events['workflow/agent-start'] by `agent.seq`, exactly once per started call on every stop path — on an engine termination path (a worker killed past its grace) the end is engine-synthesized with outcome `'cancelled'`.

```ts cordis-catalog
/**
 * One `agent()` call settled (clean result, child failure, or run
 * cancellation). Paired with {@link Events['workflow/agent-start']} by
 * `agent.seq`, exactly once per started call on every stop path — on an
 * engine termination path (a worker killed past its grace) the end is
 * engine-synthesized with outcome `'cancelled'`.
 * @param info - the run's identity snapshot.
 * @param agent - the call identity plus its outcome.
 * @mode emit
 */
'workflow/agent-end'(info: WorkflowRunInfo, agent: WorkflowAgentEndInfo): void
```

Source: [`packages/workflow/workflow/src/index.ts`](../../packages/workflow/workflow/src/index.ts)

<a id="workflowagent-start--emit"></a>

#### `workflow/agent-start` — emit

One `agent()` call established a published child run. Paired with Events['workflow/agent-end'] by `agent.seq`. A call that never receives a published run from the provider emits neither event in this pair.

```ts cordis-catalog
/**
 * One `agent()` call established a published child run. Paired with
 * {@link Events['workflow/agent-end']} by `agent.seq`. A call that never
 * receives a published run from the provider emits neither
 * event in this pair.
 * @param info - the run's identity snapshot.
 * @param agent - the call's sequence number, label, phase, and child id.
 * @mode emit
 */
'workflow/agent-start'(info: WorkflowRunInfo, agent: WorkflowAgentInfo): void
```

Source: [`packages/workflow/workflow/src/index.ts`](../../packages/workflow/workflow/src/index.ts)

<a id="workflowend--emit"></a>

#### `workflow/end` — emit

A workflow run settled (any stop reason). Fired when WorkflowRun.result resolves. Paired with Events['workflow/start'].

```ts cordis-catalog
/**
 * A workflow run settled (any stop reason). Fired when
 * {@link WorkflowRun.result} resolves. Paired with
 * {@link Events['workflow/start']}.
 * @param info - the run's identity snapshot.
 * @param result - the outcome data (stop reason, error, agent count) —
 *   deliberately WITHOUT the result value (see {@link WorkflowResultInfo}).
 * @mode emit
 */
'workflow/end'(info: WorkflowRunInfo, result: WorkflowResultInfo): void
```

Source: [`packages/workflow/workflow/src/index.ts`](../../packages/workflow/workflow/src/index.ts)

<a id="workflowlog--emit"></a>

#### `workflow/log` — emit

The script emitted a narration line (a `log(message)` call).

```ts cordis-catalog
/**
 * The script emitted a narration line (a `log(message)` call).
 * @param info - the run's identity snapshot.
 * @param message - the logged message, verbatim.
 * @mode emit
 */
'workflow/log'(info: WorkflowRunInfo, message: string): void
```

Source: [`packages/workflow/workflow/src/index.ts`](../../packages/workflow/workflow/src/index.ts)

<a id="workflowphase--emit"></a>

#### `workflow/phase` — emit

The script entered a phase (a `phase(title)` call) — progress grouping for observers; no execution semantics.

```ts cordis-catalog
/**
 * The script entered a phase (a `phase(title)` call) — progress grouping
 * for observers; no execution semantics.
 * @param info - the run's identity snapshot.
 * @param title - the phase title, verbatim.
 * @mode emit
 */
'workflow/phase'(info: WorkflowRunInfo, title: string): void
```

Source: [`packages/workflow/workflow/src/index.ts`](../../packages/workflow/workflow/src/index.ts)

<a id="workflowstart--emit"></a>

#### `workflow/start` — emit

A workflow run started — the script's meta block validated, the body about to execute. Paired with Events['workflow/end'].

```ts cordis-catalog
/**
 * A workflow run started — the script's meta block validated, the body
 * about to execute. Paired with {@link Events['workflow/end']}.
 * @param info - the run's identity snapshot (id + meta).
 * @mode emit
 */
'workflow/start'(info: WorkflowRunInfo): void
```

Source: [`packages/workflow/workflow/src/index.ts`](../../packages/workflow/workflow/src/index.ts)
<!-- END GENERATED cordis-surface -->
