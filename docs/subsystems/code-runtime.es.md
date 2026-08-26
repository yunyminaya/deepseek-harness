# Runtime de código

[English](code-runtime.md) | Español

El seam de ejecución de código — un [capability seam](../../.agents/notes/implemented/architecture/2026-06-13-capability-seams.es.md) cuya Service Definition ([dsh-code-runtime](../../packages/code-runtime/code-runtime), `ctx.codeRuntime`) ejecuta un programa escrito por el modelo contra bindings asíncronos proporcionados por el host e informa de qué imprimió y qué devolvió. La ejecución de código es **una capacidad opcional**, no parte de la espina dorsal del agent loop — por eso su vocabulario vive aquí y no en [core.md](core.es.md). Los backends se diferencian por el sustrato de ejecución y el lenguaje de origen, ambos descriptores de solo lectura del servicio; el Service Provider de worker-thread y el Consumer del registro de herramientas quedan especificados por la [base de Code Mode](../../.agents/notes/implemented/feature/2026-06-15-code-mode.es.md) y el [contrato de retorno tipado](../../.agents/notes/implemented/feature/2026-07-20-code-mode-typed-tool-returns.es.md).

Fuente: [`packages/code-runtime/code-runtime/src/types.ts`](../../packages/code-runtime/code-runtime/src/types.ts)

## La ejecución: entra la solicitud, sale el resultado

Un `CodeRunRequest` lleva **todo aquello sobre lo que actúa el runtime** — según la regla de «explícito > implícito en los límites de los paquetes», los valores por defecto (presupuestos de tiempo, topes de salida) son configuración validada de la implementación, nunca un `??` oculto dentro de `run()`:

```ts type-equiv
/**
 * One run: the program source plus everything the runtime acts on. Per the
 * explicit-over-implicit convention, defaulting (time budgets, output caps)
 * is the implementation's validated config — a request carries no optional
 * tuning knobs for a hidden `??` to fill in.
 */
interface CodeRunRequest {
  /**
   * The program source, in the runtime's {@link ../index.ts | language}. It
   * runs as the body of an async function: top-level `await` and `return`
   * are available, and the completion value becomes
   * {@link CodeRunResult.value}.
   */
  program: string
  /** Host functions exposed to the program, one global object per namespace. */
  bindings: CodeBindingNamespace[]
  /**
   * Abort the run: the runtime stops the program (hard, even mid-loop) and
   * resolves with a {@link CodeRunFailure} of kind `'abort'`. In-flight
   * binding calls are the CALLER's to settle — the runtime only stops asking.
   */
  signal?: AbortSignal
}
```

El resultado informa del error como un **campo**, nunca como un rechazo de `run()` — informar de un programa fallido es trabajo del llamador, no una vía de excepción (coincide con el contrato de resolución-ante-fallo de `ShellExecutor.run`):

```ts type-equiv
/**
 * The outcome of one run. An error is a FIELD on a resolved result, never a
 * rejection of `run()` — reporting a failed program is the caller's job, not
 * an exception path.
 */
interface CodeRunResult {
  /**
   * The program's completion value (its top-level `return`), when it ran to
   * completion and the value crossed the runtime's lossless-JSON boundary.
   * Invalid or over-limit completions fail the run instead of substituting a
   * rendered string; a failed or value-less run leaves this absent.
   */
  value?: CodeJsonValue
  /** Text the program emitted, in order, bounded only as part of the outer result. */
  logs: string[]
  /** Present iff the run failed; see {@link CodeRunFailure} for the taxonomy. */
  error?: CodeRunFailure
}
```

## Bindings: funciones del host como globales del programa

Cada `CodeBindingNamespace` se convierte en un objeto global de callables asíncronos dentro del programa (el consumer de Code Mode pasa uno: `tools`). Los argumentos y las resoluciones deben ser JSON sin pérdida y cruzar sin un tope de bytes a nivel de seam; el runtime puede puentearlos mediante structured clone. Un namespace puede declarar una clase de error visible para el programa sin que el runtime conozca los nombres del consumer: el runtime inyecta el constructor real y convierte las llamadas rechazadas en instancias suyas. Un runtime trata además los nombres de binding como entrada hostil (`__proto__` es una propiedad propia ordinaria, nunca una colisión de prototipo):

```ts type-equiv
/**
 * Program-visible typed rejection for one binding namespace. The runtime
 * injects a real error constructor under `name`; rejected member calls become
 * its instances and expose the exact member name through
 * `memberNameProperty`. Both strings are runtime data rather than knowledge
 * of a particular consumer such as Code Mode.
 */
interface CodeBindingErrorClass {
  /** Constructor global and resulting `Error.name`; same portable identifier rule as {@link CodeBindingNamespace.global}. */
  name: string
  /**
   * Non-empty own property for the member name. The portable exclusion set is
   * `RESERVED_ERROR_MEMBERS` plus dunder-form names (`__x__`, non-empty
   * middle), enforced identically by every backend; any other name —
   * identifiers or not — is accepted everywhere.
   */
  memberNameProperty: string
}
```

```ts type-equiv
/**
 * A named group of {@link CodeBindingFunction}s the runtime exposes to the
 * program as one global object (e.g. `tools`). Function names are arbitrary
 * strings — a runtime must treat names like `__proto__` or `constructor` as
 * ordinary own properties (null-prototype construction), never as prototype
 * collisions.
 */
interface CodeBindingNamespace {
  /**
   * The global identifier the program sees. Must match the LANGUAGE-PORTABLE
   * identifier subset `[A-Za-z_][A-Za-z0-9_]*` and no language's reserved
   * words, so the same namespace list works against every backend regardless
   * of `language` — a JS-only spelling like `$tools` is rejected by design,
   * not just by the Python backend. Names that satisfy the identifier rule but
   * name a backend-owned slot (`RESERVED_BINDING_GLOBALS`, e.g. `console`,
   * `__dsh_main__`) are also refused everywhere; see its declaration for the
   * exact set and why each entry is reserved.
   */
  global: string
  /** The callable members, keyed by the exact name the program calls. */
  functions: Record<string, CodeBindingFunction>
  /** Optional program-visible typed rejection contract for this namespace. */
  errorClass?: CodeBindingErrorClass
}
```

```ts type-equiv
/** A lossless JSON value transferable through the dependency-light Service Definition. */
type CodeJsonValue = null | boolean | number | string | CodeJsonValue[] | { [key: string]: CodeJsonValue }
```

```ts type-equiv
/**
 * One host-side function exposed to the program as an async callable. The
 * runtime bridges calls to it (possibly across a serialization boundary), so
 * `args` and the resolution value MUST be lossless JSON. A runtime rejects a
 * lossy or non-cloneable value with a descriptive error rather than corrupting
 * the run. No seam-level byte cap applies to a binding resolution. A rejection
 * of this function surfaces inside the program as a rejection of the
 * corresponding call.
 */
type CodeBindingFunction = (args: unknown) => Promise<CodeJsonValue>
```

## Salida capturada y taxonomía de fallos

Los logs son cadenas simples en orden de emisión. El runtime captura la salida de consola y de stream del programa, pero los metadatos de canal y de método de consola no forman parte del seam porque los consumers representan solo el texto. Las implementaciones limitan el array de logs externo serializado más el payload del valor de finalización o del mensaje de fallo; la sintaxis fija del envoltorio de resultado y los espacios en blanco de presentación del consumer no forman parte de ese registro de payload variable. El desbordamiento es un fallo explícito, no una sustitución de valor in-band.

Los tipos de fallo son **resultados ortogonales informados de forma independiente** (según [defensive-patterns](../defensive-patterns.es.md)): la expiración de un presupuesto no es una excepción, un abort no es un timeout y la muerte del sustrato (p. ej. OOM) no es ninguna de las dos:

```ts type-equiv
/**
 * Why a run failed. The kinds are orthogonal outcomes reported independently
 * (per docs/defensive-patterns.md): a budget expiry is not an exception, an
 * abort is not a timeout, and a substrate death is neither.
 *
 * - `'exception'` — the program threw or failed to parse/transform.
 * - `'timeout'` — an implementation-owned budget expired; the message says which.
 * - `'abort'` — {@link CodeRunRequest.signal} fired.
 * - `'worker-exit'` — the execution substrate died without settling (e.g. OOM).
 * - `'invalid-output'` — the completion value was not lossless JSON.
 * - `'output-limit'` — the serialized outer logs/value/diagnostic exceeded the configured cap.
 */
interface CodeRunFailure {
  /** The failure class (see the interface doc for each kind's meaning). */
  kind: 'exception' | 'timeout' | 'abort' | 'worker-exit' | 'invalid-output' | 'output-limit'
  /** Human-readable detail, suitable for feeding back to a model to self-correct. */
  message: string
}
```

## El servicio

`CodeRuntime` (`ctx.codeRuntime`, abstracto — definido en [`packages/code-runtime/code-runtime/src/index.ts`](../../packages/code-runtime/code-runtime/src/index.ts)) es `run(request)` más dos descriptores de solo lectura: `language` (el lenguaje en el que debe escribirse el programa — `'typescript'` y `'python'` son los valores conocidos, los que presenta `dsh-tools`, y solo `'typescript'` tiene un backend publicado; un consumer que genera presentación específica de lenguaje conmuta según este y falla ruidosamente ante uno que no puede presentar) e `isolation` (el sustrato de ejecución — `'worker-thread'`, `'process'`, `'container'`; una etiqueta diagnóstica, **no una afirmación de seguridad**). Las implementaciones deben mantener las ejecuciones aisladas entre sí (sin estado entre ejecuciones) y disponerse hasta la quietud: las ejecuciones en curso se terminan y se espera su finalización antes de que el teardown se complete.

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.es.md).

<a id="ctxcoderuntime--coderuntime-abstract-seam"></a>

### `ctx.codeRuntime` — `CodeRuntime` (abstract seam)

Registers one `ctx.codeRuntime` implementation. Program, budget, abort, and substrate failures resolve in CodeRunResult; only Service Definition contract misuse rejects. Implementations bridge structured-cloneable bindings, materialize each declared namespace rejection class, treat programs as hostile peers, isolate runs from one another, and terminate and await in-flight runs during disposal.

```ts cordis-catalog
/**
 * Execute one program against the request's bindings and capture what it
 * emitted. See the class doc for the resolution contract (error is a result
 * field; rejection means Service Definition contract misuse only).
 * @param request - the program, its bindings, and the abort signal; the
 *   request carries everything the runtime acts on, with no hidden defaults.
 * @returns the run's outcome: completion value (when transferable), the
 *   ordered log capture, and the failure (if any).
 */
abstract run(request: CodeRunRequest): Promise<CodeRunResult>
```

Source: [`packages/code-runtime/code-runtime/src/index.ts`](../../packages/code-runtime/code-runtime/src/index.ts)
<!-- END GENERATED cordis-surface -->
