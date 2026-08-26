# Herramientas

[English](tools.md) | Español

El pipeline (canal de procesamiento) de herramientas de [dsh-tools](../../packages/core/tools). [core.md](core.es.md) presenta `ToolDefinition`, el tipo con el que se escriben los pipelines y que comparten los paquetes del núcleo; el tipo wire `ToolSchema` orientado al modelo se declara junto con la petición del modelo. Esta página documenta todos los campos de `ToolDefinition`, el DSL de schema tipado que lo construye, los tipos de ejecución con guards y los tipos de presentación en la UI.

Fuente: [`packages/core/tools/src/index.ts`](../../packages/core/tools/src/index.ts) · [`packages/core/tools/src/schema.ts`](../../packages/core/tools/src/schema.ts) · [`packages/core/tools/src/presentation.ts`](../../packages/core/tools/src/presentation.ts)

## `ToolDefinition` — una herramienta registrada

Un `ToolSchema` (los campos orientados al modelo) más una declaración de salida canónica obligatoria, la función `execute`, metadatos de planificación solo del host, un callback opcional de contenido final y presentadores de UI opcionales. El registro los alberga; el loop despacha las llamadas a través de ellos. `schemas()` del registro construye el `ToolSchema[]` orientado al modelo mediante una lista de permitidos explícita: `output`/`execute`/`finalizeContent`/`timeoutMs`/`isConcurrencySafe`/`presentCall`/`presentResult` nunca deben filtrarse a una petición del modelo.

```ts type-equiv
/** Tool-owned canonical output contract used after the body returns a JSON value. */
interface ToolOutputDefinition {
  /** Raw supported JSON Schema enforced against every successful canonical value. */
  readonly schema: JsonSchemaNode
  /** Pure projection from validated arguments and value to Native/model content. */
  render(args: unknown, value: JsonValue): ContentBlock[]
  /** Pure replayable presentation projection, computed only for top-level calls. */
  presentationMeta?(args: unknown, value: JsonValue): JsonValue
}
```

```ts type-equiv
/** A registered tool: its schema plus the execution function. */
interface ToolDefinition extends ToolSchema {
  /** Mandatory canonical output declaration. */
  readonly output: ToolOutputDefinition
  /**
   * Run one accepted call and return only its canonical lossless-JSON value.
   * Async work must observe or forward `exec.signal` and settle only after its
   * owned work reaches quiescence. The registry preserves caller cancellation
   * through around-dispatch signal replacement and does not abandon this
   * promise, but it cannot hard-kill same-process code.
   * @param args - losslessly snapshotted, frozen model arguments.
   * @param exec - execution identity, cancellation signal, and context deferral.
   * @returns the canonical value declared by `output.schema`.
   */
  execute(args: unknown, exec: ToolRunContext): Promise<unknown>
  /**
   * Synchronous last-mile transform for model-facing content. The registry
   * snapshots this callback when execution starts and invokes it exactly once
   * for every normalized outcome, including pipeline failures that bypass
   * `tools/post-execute`, immediately before lossless materialization.
   * Returning `undefined` preserves the content; every other result field
   * remains registry-owned. The callback must be total and must not throw.
   * @param exec - immutable execution identity and arguments.
   * @param result - complete normalized outcome before materialization.
   * @returns replacement content, or `undefined` to preserve it.
   */
  finalizeContent?(exec: Readonly<ToolExecution>, result: Readonly<ToolExecutionResult>): ContentBlock[] | undefined
  /**
   * Cooperative tool-call timeout budget in milliseconds. Omit for no deadline.
   * Enforced by `@deepseek-ai/dsh-tool-call-timeout-policy` (a `tools/execute` wrapper); it
   * is NEVER sent to the model — `schemas()` whitelists only name/description/
   * parameters. Declaring it asserts this tool forwards `exec.signal` to a
   * cooperative implementation that can reach quiescence when the signal aborts.
   */
  timeoutMs?: number
  /**
   * Pure synchronous classifier for overlap with sibling tool calls. Only
   * `true` opts in; omission, exceptions, non-`true` returns, and invalid
   * `defineTool` arguments are exclusive. This metadata is never model-visible.
   *
   * Opted-in executions must not mutate parent-owned state. Shared state must
   * tolerate concurrent dispatch; recorder races are permitted only when they
   * commute or fail closed. See the
   * [parallel-tool-call Agent Note](../../../../.agents/notes/implemented/feature/2026-07-10-parallel-tool-call-execution.md)
   * for the full contract.
   * @param args - parsed arguments; `defineTool` validates before calling.
   * @returns Whether this call may join a parallel group.
   */
  isConcurrencySafe?(args: unknown): boolean
  /**
   * Optional: how to present the PENDING state of one call in a UI, derived from
   * the call's `args` (parsed arguments, `unknown` — the tool validates/narrows
   * its own input). Returns a {@link ToolCallView} (a `card`-tagged render intent),
   * or `undefined` (or omit the method) to fall back to a generic presentation
   * (title = tool name, raw args as input). Pure and side-effect-free: a UI may
   * call it during live streaming AND a session-log replay, so it must depend
   * only on `args`.
   */
  presentCall?(args: unknown): ToolCallView | undefined
  /**
   * Optional: how to present the COMPLETED state, given the same `args` and the
   * durable result projection (`content`, failure state, and optional `meta`). Returns a
   * {@link ToolResultView}, or `undefined` (or omit the method) to keep the
   * pending title and render the raw result content. Pure and side-effect-free
   * for the same replay reason.
   */
  presentResult?(args: unknown, result: ToolResult): ToolResultView | undefined
}
```

`execute` recibe `args: unknown`: un `ToolDefinition` en bruto valida su propia entrada. Las herramientas de primera parte no lo escriben a mano; usan `defineTool`, que valida y estrecha los argumentos, infiere el valor de retorno del cuerpo a partir de `output.schema` y tipa ambos proyectores de salida. `finalizeContent` recibe deliberadamente la ejecución inmutable en lugar de argumentos tipados, porque también le llegan los fallos de entrada inválida y del pipeline exterior; puede imponer un límite de contenido propio de la herramienta sin perder `isError`, el valor canónico, la identidad del error estructurado, los contextos diferidos ni los metadatos de presentación.

## El DSL unificado de schema de valores JSON

Los autores de plugins usan un único vocabulario para los parámetros tipados y los valores de salida tipados. `ValueSchemaSpec` admite `string`, `number`, `integer`, `boolean`, `null`, `array`, `object`, `json` solo para el autor y `oneOf` de exactamente uno; los valores escalares de `enum` y `const` deben coincidir con el tipo de su nodo. Un nodo de objeto explícito siempre declara `additionalProperties: true | false`. Las definiciones de parámetros siguen siendo un mapa de propiedades de objeto abierto implícito, con `required: true` adjunto a cada propiedad obligatoria.

Fuente: [`packages/core/tools/src/schema.ts`](../../packages/core/tools/src/schema.ts)

```ts type-equiv
/** One author-facing schema for any lossless JSON value root. */
type ValueSchemaSpec =
  | StringValueSchemaSpec
  | NumberValueSchemaSpec
  | IntegerValueSchemaSpec
  | BooleanValueSchemaSpec
  | NullValueSchemaSpec
  | ArrayValueSchemaSpec
  | ObjectValueSchemaSpec
  | JsonValueSchemaSpec
  | OneOfValueSchemaSpec
```

```ts type-equiv
/** One implicit parameter-root property, optionally required. */
type ParameterPropertySpec = ValueSchemaSpec & { required?: true }
```

```ts type-equiv
/**
 * Tool parameter schema. The map itself is an implicit open object root;
 * requiredness remains a per-property `required: true` annotation.
 */
type ParameterSchemaSpec = {
  [key: string]: ParameterPropertySpec
  [key: symbol]: never
}
```

`{ type: 'json' }` infiere `JsonValue` y compila a un schema en bruto sin restricciones, solo de anotación. Las raíces de salida pueden ser objetos, arrays, escalares o null. `InferValue<S>` respeta las restricciones literales y la apertura del objeto hasta 16 niveles de contenedor y luego recurre a `JsonValue` en lugar de agotar la pila de instanciación de tipos de TypeScript. `InferArgs<P>` convierte la obligatoriedad por propiedad en claves de cadena obligatorias y opcionales:

```ts type-equiv
/**
 * Infer the TypeScript value accepted by an author-facing value schema. Exact
 * inference is bounded to 16 container levels, then falls back to `JsonValue`.
 */
type InferValue<S> = InferValueAt<S, []>
```

```ts type-equiv
/** Infer the TypeScript argument object for an implicit parameter schema. */
type InferArgs<S> = InferProperties<S, []>
```

`defineTool({ name, description, parameters, output, execute, … })` vincula la inferencia de parámetros a `parameterSchemaSpecToJsonSchema()` y `validateArgs()`, y vincula `execute`/`render`/`presentationMeta` a `InferValue<OutputSchema>`. Los registros de schema contienen solo claves de cadena propias y enumerables, y los arrays de schema son arrays intrínsecos densos, de modo que la inferencia, la compilación y la validación observan la misma declaración. La inferencia se mantiene exacta hasta 16 niveles de contenedor y luego se amplía a `JsonValue`; la validación en tiempo de ejecución sigue recorriendo el schema completo. `valueSchemaSpecToJsonSchema()` compila las declaraciones de salida a través del mismo subconjunto en bruto impuesto. Un desajuste de parámetros lanza `ToolArgsError` (`INVALID_ARGS`); un cuerpo o un valor posterior a la política inválidos lanzan `ToolOutputError` (`INVALID_TOOL_OUTPUT`). Ambos usan la vía normal de errores de herramienta. El JSON Schema en bruto sigue abierto por defecto; las palabras clave no admitidas se rechazan en lugar de aceptarse sin aplicación.

El registro es un contrato de confianza dentro del mismo proceso. El registro toma prestada la definición tipada como entrada de solo lectura, exige `output`, valida su schema en bruto y comprueba requisitos semánticos como un `timeoutMs` finito positivo; `schemas()` construye la proyección orientada al modelo al componer una petición, de modo que la ejecución y la presentación comparten una única definición resuelta sin filtrar callbacks por el cable.

## `ToolRestriction` — el filtro en vivo de un ámbito sobre lo que hereda

`ToolRestriction` se aplica a las herramientas que un ámbito hereda: la capa global del despliegue más cada ámbito ancestro de su cadena. El registro compila los nombres de solo lectura en conjuntos privados, interseca varias restricciones y luego superpone los registros PROPIOS del ámbito, que siguen exentos para que un hijo delegado conserve las herramientas a través de las que responde. Un filtro solo de denegación admite las herramientas heredadas no listadas posteriores, mientras que una lista de permitidos las excluye.

```ts type-equiv
/**
 * Per-scope filter over global tools. Restrictions intersect and do not affect
 * scoped registrations or the reserved Code Mode transport.
 */
interface ToolRestriction {
  /** Global tool names that stay visible; everything else is removed. */
  readonly allow?: readonly string[]
  /** Global tool names removed from visibility. */
  readonly deny?: readonly string[]
}
```

## Ejecución: waterfalls extensibles más política monótona

`ctx.tools.execute()` acepta un `ToolExecutionInput` propiedad del llamador con un `signal` de solo lectura obligatorio, materializa sus argumentos JSON parseados una sola vez en un `ToolExecution` propiedad del pipeline y ejecuta esa llamada a través de `tools/pre-execute` (el waterfall reordenable de allow/deny/ask) → guards monótonos registrados → `tools/execute` (envoltorios alrededor del despacho) → `tools/post-execute` (inspeccionar/sustituir el resultado) → el `finalizeContent` opcional propio de la definición → `tools/result` (el resultado autoritativo inmutable). Solo la vista de `tools/execute` puede sustituir el signal obligatorio. El resultado es un `ToolExecutionResult`.

```ts type-equiv
/** Opaque call identity that permits correlation without exposing mutable execution state. */
type ToolExecutionToken = symbol & { readonly [toolExecutionTokenBrand]: true }
```

```ts type-equiv
/**
 * Caller-supplied description of one tool call. {@link ToolRuntime.execute}
 * adds the registry-owned token to form a pipeline {@link ToolExecution};
 * callers do not choose that token.
 */
interface ToolExecutionInput {
  readonly callId: CallId
  /**
   * Root model-requested call owning this execution tree. Callers omit it for
   * a root execution; nested dispatchers propagate the enclosing value.
   */
  readonly rootCallId?: CallId
  readonly name: string
  /** Losslessly JSON-serializable parsed arguments (tools validate their own schema). */
  readonly arguments: unknown
  /** The agent on whose behalf the call runs (set by the agent loop). */
  readonly agent?: Agent
  /**
   * Opaque token of the enclosing transport execution, when one exists. Code
   * Mode sets this on SDK sub-dispatches so commit-style observers can wait for
   * the outer `run_code` outcome without receiving its live mutable execution.
   * The token also marks the call as a transport sub-dispatch rather than a
   * model-direct call: under `mode: 'code'`, only calls WITH a parent may
   * execute a native tool name — a model-direct call (no parent) is denied as
   * `UNKNOWN_TOOL` before the policy pipeline. See {@link ToolRuntime.execute}.
   */
  readonly parent?: ToolExecutionToken
  /** Required caller-owned cancellation for this invocation. */
  readonly signal: AbortSignal
}
```

Un cuerpo de herramienta recibe la extensión de tiempo de ejecución. `deferContext()` adjunta contexto al resultado propio de la ejecución — el canal de despacho anidado de las herramientas compuestas, utilizable también por una herramienta hoja que acuñe una instrucción procedente de un plugin — sin inyectar dentro de la llamada exterior aún abierta.

```ts type-equiv
/**
 * Runtime context handed to a tool implementation after the registry has
 * accepted a {@link ToolExecution}. {@link deferContext} attaches context to
 * this execution's own result — a composite tool ferries nested-dispatch
 * context back to the outer result, and a leaf tool may mint a fresh
 * plugin-sourced instruction; the loop appends it only after the
 * `tool/result`.
 */
interface ToolRunContext extends ToolExecution {
  /**
   * Defer one context — typically a nested-dispatch context ferried by a
   * composite tool, or a fresh plugin-sourced instruction — until this tool's
   * final result reaches the agent loop. Contexts retain their individual
   * source and metadata and are emitted in call order.
   */
  deferContext(context: UserMessage): void
  /**
   * Mark a successful final result as terminal for the current agent turn.
   * The marker rides this execution's own result (`concludesTurn` exists only
   * on {@link ToolExecutionSuccess}); a composite that dispatches nested
   * calls forwards it from the nested result, exactly like
   * `additionalContexts`, so only an authoritative nested success can
   * conclude the enclosing run.
   */
  concludeTurn(): void
}
```

El agent loop pregunta al registro por el modo de ejecución de cada llamada pendiente y lo usa para formar barreras exclusivas y ejecuciones paralelas en grupo rodante:

```ts type-equiv
/**
 * Scheduling mode for one pending call. `parallel` may overlap with siblings;
 * `exclusive` runs alone and forms an ordering barrier.
 */
type ToolExecutionMode =
  | { kind: 'parallel' }
  | { kind: 'exclusive' }
```

El puente de Code Mode además expone cada subdespacho resuelto al waterfall `tools/code-dispatch-log`, que puede cambiar la copia del contenido del evento duradero (el valor del programa y el resultado visible para el modelo permanecen intactos):

```ts type-equiv
/**
 * One settled `run_code` sub-dispatch about to be logged, as seen by the
 * `tools/code-dispatch-log` waterfall: the parent execution (session owner,
 * outer call identity), the sub-call identity, and the outcome whose durable
 * copy a listener may reshape. `content` is the RENDERED result projection
 * (what a native `tool/result` would carry) — the program itself received
 * the structured `value` (or just the error message on failure); only the
 * `tool/code-dispatch` event's copy changes.
 */
interface CodeDispatchLog {
  /** The outer `run_code` execution. */
  readonly exec: ToolExecution
  /** The calling agent (the scope routing key and the spill owner), when the outer call has one. */
  readonly agent?: Agent
  /** Deterministic sub-call id (`<parent>:code:<n>`). */
  readonly subCallId: CallId
  /** The dispatched sub-tool name. */
  readonly name: string
  /** Whether the sub-call settled as an error. */
  readonly isError: boolean
  /** The sub-call's complete model-facing content (the settle event's default payload). */
  readonly content: ContentBlock[]
}
```

```ts type-equiv
/**
 * One pending tool call inside the registry pipeline. Parsed arguments cross
 * one lossless-JSON materialization boundary before policy and are deep-frozen;
 * call identity, the caller signal, and the registry-assigned {@link token} are
 * readonly. The registry freezes the complete object before `tools/result`
 * observers run.
 */
interface ToolExecution extends ToolExecutionInput {
  /** Root model-requested call, resolved for every root and nested execution. */
  readonly rootCallId: CallId
  /** Registry-assigned identity shared with nested calls only as their opaque `parent` token. */
  readonly token: ToolExecutionToken
}
```

```ts type-equiv
/**
 * Around-dispatch view of a {@link ToolExecution}. A `tools/execute` wrapper
 * may replace the signal for its delegated lifetime, but it cannot remove it.
 * The registry fuses every replacement with the captured caller signal.
 */
interface ToolDispatchExecution extends Omit<ToolExecution, 'signal'> {
  /** Cancellation signal visible to the next wrapper or tool body. */
  signal: AbortSignal
}
```

`ToolExecutionToken` es un `Symbol` opaco de tiempo de ejecución que solo se usa para comparar identidades. Antes de la política, `execute()` materializa y congela los argumentos, rechaza la entrada que no sea JSON y asigna el token. Los campos de identidad, el signal obligatorio del llamador y el token de padre opcional siguen siendo de solo lectura. Un envoltorio `ToolDispatchExecution` puede sustituir el signal, pero no eliminarlo; el registro vuelve a fusionar el signal del llamador antes de invocar el cuerpo. Los observadores finales reciben la identidad de ejecución congelada.

Un `ToolGuard` es la política final previa al despacho, consciente del ámbito. Su tipo de retorno no tiene deliberadamente ningún resultado de allow: `undefined` conserva la decisión del waterfall, mientras que una razón devuelta solo puede reducir el permiso, de modo que un oyente posterior no puede deshacerla.

```ts type-equiv
/**
 * A monotonic execution guard evaluated after every `tools/pre-execute`
 * listener and before the tool body. Returning a reason denies the call;
 * returning `undefined` leaves it unchanged. Because guards have no allow
 * result, listener ordering cannot turn a denial back into permission.
 * @param execution - the identity-protected call after extensible pre-execute policy completed.
 * @returns a final denial reason, or `undefined` to leave the call allowed.
 */
type ToolGuard = (execution: Readonly<ToolExecution>) => string | undefined
```

```ts type-equiv
/** Canonical failure detail; internal routing information remains optional. */
interface ToolFailure {
  /** Human-readable failure message without the Native `Error: ` envelope. */
  message: string
  /** Internal error class/code used by policy and durable diagnostics. */
  info?: ToolErrorInfo
}
```

```ts type-equiv
/** Successful canonical tool execution, including its Native/model projection. */
interface ToolExecutionSuccess {
  readonly isError: false
  /** Execution-local canonical value; deliberately omitted from durable events. */
  readonly value: JsonValue
  readonly content: ContentBlock[]
  readonly error?: never
  readonly meta?: JsonValue
  readonly additionalContexts?: UserMessage[]
  /** The agent loop stops after committing this successful result batch. */
  readonly concludesTurn?: true
}
```

```ts type-equiv
/** Failed canonical tool execution; failures never carry a successful value. */
interface ToolExecutionFailure {
  readonly isError: true
  readonly error: ToolFailure
  readonly value?: never
  readonly content: ContentBlock[]
  readonly meta?: JsonValue
  readonly additionalContexts?: UserMessage[]
  readonly concludesTurn?: never
}
```

```ts type-equiv
/** The discriminated, execution-local outcome of one tool call. */
type ToolExecutionResult = ToolExecutionSuccess | ToolExecutionFailure
```

El resultado solo transporta el desenlace. La identidad de la llamada permanece en el `ToolExecution` inmutable que lo acompaña por cada hook y en los eventos de sesión duraderos `tool/call` / `tool/result`, de modo que los envoltorios no pueden crear una segunda identidad discrepante. El `value` canónico es local a la ejecución: el loop persiste solo `content`, `error` y `meta`, mientras que `tool/code-dispatch` almacena verbatim el `content` renderizado y el `isError` de la subllamada. La reproducción reproduce la presentación, pero no puede reconstruir los valores intermedios canónicos.

En caso de éxito, el registro toma una instantánea del valor del cuerpo, lo valida, lo congela e invoca el renderizador puro más el proyector opcional de metadatos de la llamada de nivel superior. Además materializa por separado los campos de presentación duraderos inmediatamente antes de `tools/result`; un valor inválido, un fallo del renderizador/proyector o una presentación que no sea JSON se convierten en un `isError` seguro para JSON. El observador final en vivo ve, por tanto, el valor exacto local a la ejecución junto a los campos seguros para el anexo duradero posterior.

Antes del contenido final, el registro materializa el resultado candidato; un fallo en el contenido, el error estructurado, el contexto adicional o los metadatos de presentación se convierten en un resultado `isError` seguro para JSON que aun así llega a `finalizeContent`. El registro invoca ese callback exactamente una vez y luego materializa y congela el resultado aceptado inmediatamente antes de `tools/result`, de modo que el desenlace observado en vivo es seguro para el anexo duradero posterior de `tool/result`.

Cada waterfall de intercepción devuelve una **Decisión** tipada (el modismo compartido con los waterfalls de `agent/*`). Los oyentes de `tools/pre-execute` reciben `(exec, next)` y devuelven una `PreToolDecision`; los envoltorios de `tools/execute` devuelven un `ToolExecutionResult`; los oyentes de `tools/post-execute` reciben `(exec, result, next)` y devuelven una `PostToolDecision`:

```ts type-equiv
/**
 * Pre-dispatch decision. `allow` runs the call; `deny` materializes an error;
 * `ask` runs only after an approval service returns `allowed-once` and otherwise
 * denies. Input rewriting is excluded because arguments are already logged and
 * presented.
 */
type PreToolDecision =
  | { kind: 'allow' }
  | { kind: 'deny'; reason: string }
  | { kind: 'ask'; reason?: string }
```

```ts type-equiv
/**
 * Post-dispatch decision: accept, replace one projection, attach context for the
 * next request, or block by turning corrective feedback into an error result.
 */
type PostToolDecision =
  | { kind: 'accept'; content?: ContentBlock[]; value?: never; additionalContexts?: UserMessage[] }
  | { kind: 'accept'; value: JsonValue; content?: never; additionalContexts?: UserMessage[] }
  | { kind: 'block'; feedback: ContentBlock[]; additionalContexts?: UserMessage[] }
```

Llama a `next()` para el comportamiento por defecto o devuelve una decisión para cortocircuitar. La política previa puede denegar o preguntar (ask); solo `allowed-once` continúa, mientras que una no-concesión, la falta de canal o servicio de aprobación o una petición sin agent se convierte en una denegación. Los guards pueden imponer aun así una denegación final. Los argumentos no se pueden reescribir porque el historial, la auditoría, la UI y la ejecución deben coincidir.

La política posterior puede sustituir el contenido o el valor, nunca ambos. Sustituir el contenido conserva el valor canónico y los metadatos existentes; sustituir el valor se revalida y recalcula el contenido/los metadatos; un bloque elimina el valor y se convierte en un `isError` que contiene la retroalimentación correctiva. Sustituir el contenido es política de presentación, no de confidencialidad: un oyente que deba ocultar el valor programático lo bloquea o lo sustituye. `tools/result` recibe la ejecución y el resultado congelados después de la normalización; los observadores no pueden transformarlos y sus fallos quedan contenidos. Las herramientas desconocidas y las que lanzan excepciones se convierten en errores estructurados (`ToolNotFoundError` se asigna a `UNKNOWN_TOOL`), de modo que la llamada falla sin terminar el turno.

## El subconjunto impuesto de JSON Schema en bruto

Los schemas en bruto de subagentes, flujos de trabajo, MCP y registros dinámicos usan la contraparte a nivel de wire del DSL del autor. `assertSupportedJsonSchema()` acepta cualquier raíz JSON, `validateJsonSchemaValue()` la aplica y `JsonSchemaError` informa de cualquier ruta de schema no admitida o malformada. El nodo vacío solo de anotación significa JSON sin pérdidas sin restricciones. `oneOf` exige al menos dos ramas y un valor debe coincidir exactamente con una. Los consumidores que siguen necesitando una raíz de objeto llaman a `assertObjectJsonSchema()` y portan un `ObjectJsonSchema`; así la salida estructurada definida por el llamador de subagente/flujo de trabajo sigue enraizada en objetos sin restringir el vocabulario compartido.

```ts type-equiv
/** Scalar JSON values supported by `enum` and `const`. */
type JsonSchemaScalar = string | number | boolean | null
```

```ts type-equiv
/** Single-type keywords accepted by the enforced subset. */
type JsonSchemaType = 'object' | 'array' | 'string' | 'number' | 'integer' | 'boolean' | 'null'
```

```ts type-equiv
/**
 * One raw JSON Schema node in the enforced subset. The optional fields express
 * the external wire schema; {@link assertSupportedJsonSchema} rejects invalid
 * combinations before a caller treats the node as trusted.
 */
interface JsonSchemaNode {
  /** Omit with no constraints for any JSON value, or use `oneOf`. */
  type?: JsonSchemaType
  /** Exactly one branch must validate; at least two branches are required. */
  oneOf?: JsonSchemaNode[]
  /** Nested property schemas (`type: 'object'` only). */
  properties?: Record<string, JsonSchemaNode>
  /** Required property names; each must appear in `properties`. */
  required?: string[]
  /** `false` rejects undeclared keys; absent/`true` follows JSON Schema's open default. */
  additionalProperties?: boolean
  /** Item schema (`type: 'array'` only); absent accepts any JSON item. */
  items?: JsonSchemaNode
  /** Allowed values for a scalar node. */
  enum?: JsonSchemaScalar[]
  /** The single allowed value for a scalar node. */
  const?: JsonSchemaScalar
  /** Annotation, ignored for validation. */
  description?: string
  /** Annotation, ignored for validation. */
  title?: string
  /** Annotation, ignored for validation but required to be lossless JSON. */
  default?: JsonValue
  /** Annotation, ignored for validation but required to be lossless JSON. */
  examples?: JsonValue
}
```

```ts type-equiv
/** A consumer-constrained object-rooted schema. */
type ObjectJsonSchema = JsonSchemaNode & { type: 'object' }
```

## Vocabulario de UI de presentación de herramientas

Cómo quiere una herramienta que se muestre su llamada en una UI (una tarjeta de llamada de herramienta en el editor, una línea de log del CLI), neutral respecto al provider para que la herramienta se describa a sí misma sin depender de ningún protocolo de cliente. `presentCall`/`presentResult` devuelven una **intención de renderizado etiquetada como `card`** — una unión discriminada sobre la que conmuta un puente de UI:

- `ToolCallView` (pendiente): `{ card: 'generic', title, kind?, rawInput?, content?, locations? }` (la tarjeta por defecto; `locations` es un `{ path, line? }[]` de archivos que la llamada lee/modifica, para que el editor los siga), `{ card: 'terminal', title, description?, cwd? }` (un comando de shell → una tarjeta de terminal), o `{ card: 'diff', title, diffs, locations? }` (una creación/modificación de archivo → una tarjeta de diff en línea; `diffs` es un `{ path, oldText, newText }[]`, con `oldText: null` para un archivo nuevo).
- `ToolResultView` (completada): `{ card: 'generic', title?, content? }`, `{ card: 'terminal', title?, output?, exitCode?, signal? }` (la salida capturada de la ejecución y su estado de salida; una UI capaz muestra una píldora de estado de salida, mientras que otra puede derivar un respaldo cercado ` ```console `), `{ card: 'diff', title?, diffs }` (una mutación de archivo completada → el cambio que mostrar, normalmente los fragmentos (hunks) aplicados con líneas de contexto calculadas a partir del contenido anterior/posterior, o un diff de todo el archivo cuando no hay imagen anterior), `{ card: 'search', shape, title?, truncated, total, … }` (una búsqueda de descubrimiento completada → coincidencias agrupadas por archivo para `shape: 'matches'` (grep) o una lista plana de rutas para `shape: 'paths'` (glob); `truncated`/`total` informan de si el resultado en línea se recortó, para que una UI nunca presente un resultado parcial como completo; la vista no lleva texto de resultado: una UI sin tarjeta de búsqueda recurre al contenido bruto del resultado), `{ card: 'read', title?, path, offset, lines, totalLines, lang?, content? }` (una lectura de archivo completada → una vista de código numerada y opcionalmente resaltada por sintaxis; `offset` es la primera línea en base 1 que pidió la ventana, conservada incluso cuando `lines` está vacío; `lang` es una pista de idioma de la extensión y `content` es el texto sin envoltura al que recurre una UI sin soporte de lectura), o `{ card: 'web', kind: 'search' | 'fetch', title?, … }` (una recuperación web completada; `kind: 'search'` porta los `sources`/`answer?`/`truncated` estructurados, `kind: 'fetch'` porta `url`/`statusCode`/`truncated`, y una UI sin la capacidad `web` recurre al contenido bruto del resultado: el cuerpo no se duplica en la vista). Las vistas completadas sustituyen a las pendientes, de modo que las herramientas de mutación devuelven un resultado diff incluso cuando duplica el fragmento del momento de la llamada; una búsqueda y una recuperación web no tienen análogo `card` en el momento de la llamada (su estado pendiente sigue siendo una tarjeta genérica, porque el resultado estructurado solo existe después de `execute`).

`ToolCallKind` (`'read' | 'edit' | 'delete' | 'move' | 'search' | 'execute' | 'fetch' | 'other'`) elige un icono en una tarjeta genérica. `FileLocation` (`{ path, line? }`), `FileDiff` (`{ path, oldText, newText }`) y `ReadFileLine` (`{ number, text }`, una línea numerada en base 1 de una ventana de lectura) son el vocabulario compartido de tarjetas de archivo. El diseño queda fijado en [la Agent Note de la unión de intenciones de renderizado](../../.agents/notes/implemented/architecture/2026-07-02-tool-render-intent-union.es.md); los tiempos de ejecución del host/cliente proyectan este vocabulario neutral en sus propias vistas.

La documentación completa de los campos de presentación está en [`packages/core/tools/src/presentation.ts`](../../packages/core/tools/src/presentation.ts). El schema `bash` y su ejecutor están en [shell.md](shell.es.md); los controles genéricos de segundo plano, en [jobs.md](jobs.es.md).

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxtools--toolruntime"></a>

### `ctx.tools` — `ToolRuntime`

Tool registry and execution pipeline. Scoped registrations shadow globals; one visibility resolver feeds presentation, lookup, and dispatch.

```ts cordis-catalog
/**
 * Present the calling scope's tools in `mode` instead of the deployment
 * default. Nearest scope on the chain wins, so a preset's standing
 * declaration covers every agent joined under it.
 *
 * Scoped only, and one declaration per scope: this is how an agent preset
 * composes Code Mode agents beside native ones in the same process, and a
 * process-global override would be the `mode` config field instead.
 * @param mode - the presentation the covered agents' models see.
 * @returns the exact disposer that restores the deployment default.
 */
presentAs(mode: ToolPresentationMode): () => void

/**
 * Register globally or in the calling agent scope. Scoped tools shadow
 * globals; duplicates within one layer and the reserved `run_code` name fail.
 * @param definition - tool schema, execution, and optional finalization/presentation callbacks.
 * @returns the exact disposer that unregisters the tool.
 */
register(definition: ToolDefinition): () => void

/**
 * Restrict global tools for the calling agent scope. Empty filters, unknown
 * names, scope-local names, and reserved transport names fail. Restrictions
 * intersect; scoped registrations remain visible.
 * @param filter - global-tool mask: `allow` (keep only) and/or `deny` (remove).
 * @returns the exact disposer that lifts this restriction.
 */
restrict(filter: ToolRestriction): () => void

/**
 * Register a monotonic guard after the extensible `tools/pre-execute`
 * waterfall. A plain-context guard applies globally; one registered through
 * `agent.ctx` applies only to that agent. Any matching guard may deny by
 * returning a reason, while no guard can force-allow a call another guard
 * denied. The exact effect disposer is returned for ordered ownership and
 * HMR cleanup.
 * @param guard - synchronous check; a returned string denies the execution.
 * @returns the exact disposer that unregisters the guard.
 */
guard(guard: ToolGuard): () => void

/**
 * Look up a tool as one scope sees it (scoped
 * shadows global; a restricted-away global reads as absent). Presenters pass
 * the calling agent so the rendered card matches the definition that
 * actually executed.
 * @param name - the tool name as registered.
 * @param scope - the viewing scope (the agent); omitted = the global view.
 * @returns the definition the scope resolves, or undefined when none is visible.
 */
get(name: string, scope?: ScopeKey): ToolDefinition | undefined

/**
 * Project visible definitions onto the allowlisted model-facing schema fields,
 * excluding execution and presentation callbacks.
 * @param scope - the viewing scope (the agent); omitted = the global view.
 * @returns one deep-cloned schema per visible tool.
 */
schemas(scope?: ScopeKey): ToolSchema[]

/**
 * Classify a pending call through the caller's visible tool definition. Only
 * an exact `true` is parallel; unknown, hidden, undeclared, invalid, or
 * throwing classifiers are exclusive.
 * @param exec - call name, parsed arguments, and optional agent scope.
 * @returns the fail-closed scheduling mode.
 */
executionMode(exec: ToolExecutionInput): ToolExecutionMode

/**
 * Execute through pre-policy, guards, around-dispatch, post-policy,
 * definition-owned content finalization, and final notification. Tool and
 * listener failures resolve as materialized error results; an invisible tool
 * reports `UNKNOWN_TOOL`. The returned outcome is the same lossless, frozen
 * snapshot final observers receive. Cancellation
 * arriving after entry and before final result materialization skips a
 * not-yet-started body with `ABORTED_BEFORE_DISPATCH` or replaces a
 * successful started outcome with `ABORTED`; already-started work is still
 * drained and may retain a tool-owned structured error.
 * @param exec - the typed same-process call input. The registry assigns its
 *   correlation token before policy begins.
 * @returns the materialized final result.
 */
async execute(exec: ToolExecutionInput): Promise<ToolExecutionResult>
```

Types: [ScopeKey](scope.es.md)

Source: [`packages/core/tools/src/index.ts`](../../packages/core/tools/src/index.ts)

<a id="tools-events"></a>

### `tools/*` events

<a id="toolschange--emit"></a>

#### `tools/change` — emit

A tool was registered or unregistered, or a scoped restriction changed (the available tool set changed — possibly for one scope only). An UNFILTERED registry-subject notification, deliberately not scope-filtered dispatch: a global change concerns every agent's next assembly, so a scoped listener subscribing here sees every change, not just its own scope's.

```ts cordis-catalog
/**
 * A tool was registered or unregistered, or a scoped restriction changed
 * (the available tool set changed — possibly for one scope only). An
 * UNFILTERED registry-subject notification, deliberately not scope-filtered
 * dispatch: a global change concerns every agent's next assembly, so a
 * scoped listener subscribing here sees every change, not just its own
 * scope's.
 * @mode emit
 */
'tools/change'(): void
```

Source: [`packages/core/tools/src/index.ts`](../../packages/core/tools/src/index.ts)

<a id="toolscode-dispatch-log--waterfall"></a>

#### `tools/code-dispatch-log` — waterfall

Allow a listener to replace content in the DURABLE LOG COPY of one `run_code` sub-dispatch outcome before the bridge appends its `tool/code-dispatch` event. `next()` keeps the content unchanged; a listener may return replacement blocks (e.g. the spill policy's preview + locator for an oversized text result). Only the logged copy is affected — the program already received the complete value, and the model sees neither. A throwing listener is contained: the bridge falls back to logging the original settled content. Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent's dispatches.

```ts cordis-catalog
/**
 * Allow a listener to replace content in the DURABLE LOG COPY of one
 * `run_code` sub-dispatch outcome before the bridge appends its
 * `tool/code-dispatch` event. `next()` keeps the
 * content unchanged; a listener may return replacement blocks (e.g. the
 * spill policy's preview + locator for an oversized text result). Only the
 * logged copy is affected — the program already received the complete
 * value, and the model sees neither. A throwing listener is contained:
 * the bridge falls back to logging the original settled content.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent's dispatches.
 * @param dispatch - the parent execution, sub-call identity, and the settled content to log.
 * @mode waterfall
 */
'tools/code-dispatch-log'(this: Scoped<ToolRuntime>, dispatch: CodeDispatchLog, next: () => Promise<ContentBlock[]>): Promise<ContentBlock[]>
```

Types: [ContentBlock](llm-streaming.es.md) · [Scoped](scope.es.md)

Source: [`packages/core/tools/src/index.ts`](../../packages/core/tools/src/index.ts)

<a id="toolsexecute--waterfall"></a>

#### `tools/execute` — waterfall

Around-dispatch waterfall for timeout, retry, or metrics. `next()` returns a normalized result; wrappers may change only `exec.signal`, while call identity remains immutable. The registry re-fuses the original caller signal before the body, so replacement cannot detach caller cancellation; wrappers must still restore their signal and reach quiescence. Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent's calls.

```ts cordis-catalog
/**
 * Around-dispatch waterfall for timeout, retry, or metrics. `next()` returns
 * a normalized result; wrappers may change only `exec.signal`, while call
 * identity remains immutable. The registry re-fuses the original caller
 * signal before the body, so replacement cannot detach caller cancellation;
 * wrappers must still restore their signal and reach quiescence.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent's calls.
 * @param exec - the allowed call about to dispatch (name, parsed arguments, caller agent, signal).
 * @mode waterfall
 */
'tools/execute'(this: Scoped<ToolRuntime>, exec: ToolDispatchExecution, next: () => Promise<ToolExecutionResult>): Promise<ToolExecutionResult>
```

Types: [Scoped](scope.es.md)

Source: [`packages/core/tools/src/index.ts`](../../packages/core/tools/src/index.ts)

<a id="toolspost-execute--waterfall"></a>

#### `tools/post-execute` — waterfall

Accept, replace, enrich, or block a normalized dispatch result. `next()` accepts it unchanged; thrown tools still reach this waterfall as errors. Async listeners must observe `exec.signal`; after they settle, caller cancellation replaces only a successful accepted outcome with the code selected by whether the tool body was invoked. Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent's calls.

```ts cordis-catalog
/**
 * Accept, replace, enrich, or block a normalized dispatch result. `next()`
 * accepts it unchanged; thrown tools still reach this waterfall as errors. Async
 * listeners must observe `exec.signal`; after they settle, caller
 * cancellation replaces only a successful accepted outcome with the code
 * selected by whether the tool body was invoked.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent's calls.
 * @param exec - the call that just ran (name, parsed arguments, caller agent).
 * @param result - the dispatch outcome a listener may accept, replace, or block.
 * @mode waterfall
 */
'tools/post-execute'(this: Scoped<ToolRuntime>, exec: ToolExecution, result: Readonly<ToolExecutionResult>, next: () => Promise<PostToolDecision>): Promise<PostToolDecision>
```

Types: [Scoped](scope.es.md)

Source: [`packages/core/tools/src/index.ts`](../../packages/core/tools/src/index.ts)

<a id="toolspre-execute--waterfall"></a>

#### `tools/pre-execute` — waterfall

Allow, deny, or ask before dispatch. `next()` delegates to allow; missing approval support turns `ask` into denial. Async gates must observe `exec.signal`; the registry rechecks cancellation after they settle but never abandons their promise. Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent's calls.

```ts cordis-catalog
/**
 * Allow, deny, or ask before dispatch. `next()` delegates to allow; missing
 * approval support turns `ask` into denial. Async gates must observe
 * `exec.signal`; the registry rechecks cancellation after they settle but
 * never abandons their promise.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): agent-scoped listeners receive only that agent's calls.
 * @param exec - the pending call (name, parsed arguments, caller agent).
 * @mode waterfall
 */
'tools/pre-execute'(this: Scoped<ToolRuntime>, exec: ToolExecution, next: () => Promise<PreToolDecision>): Promise<PreToolDecision>
```

Types: [Scoped](scope.es.md)

Source: [`packages/core/tools/src/index.ts`](../../packages/core/tools/src/index.ts)

<a id="toolsresult--emit"></a>

#### `tools/result` — emit

Observe the frozen, lossless-JSON final outcome. Listener failures are contained. Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): keyed by `exec.agent`.

```ts cordis-catalog
/**
 * Observe the frozen, lossless-JSON final outcome. Listener failures are contained.
 * Scope-filtered dispatch (`@deepseek-ai/dsh-scope`): keyed by `exec.agent`.
 * @param exec - the execution object that traversed the pipeline.
 * @param result - a deep-frozen snapshot of the final returned result.
 * @mode emit
 */
'tools/result'(this: Scoped<ToolRuntime>, exec: Readonly<ToolExecution>, result: Readonly<ToolExecutionResult>): undefined
```

Types: [Scoped](scope.es.md)

Source: [`packages/core/tools/src/index.ts`](../../packages/core/tools/src/index.ts)
<!-- END GENERATED cordis-surface -->
