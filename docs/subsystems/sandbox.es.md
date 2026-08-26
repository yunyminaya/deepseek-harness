# Sandbox de procesos

[English](sandbox.md) | Español

El seam de sandbox de procesos de [dsh-sandbox](../../packages/sandbox/sandbox) envuelve el argv de un subproceso del mismo mundo en una política de efectos de archivo sin acoplar a los Consumer a un runner de plataforma. [dsh-sandbox-local](../../packages/sandbox/sandbox-local) aporta los backends de bwrap/Landlock para Linux, Seatbelt para macOS y token restringido de ACL de Windows; [dsh-bash-sandbox](../../packages/shell/bash-sandbox) y [dsh-pwsh-sandbox](../../packages/shell/pwsh-sandbox) lo consumen. Los contenedores, las microVM y la ejecución remota son implementaciones hermanas de seams de capacidad completos, no providers de `ctx.sandbox`.

Source: [`packages/sandbox/sandbox/src/index.ts`](../../packages/sandbox/sandbox/src/index.ts)

## Modos y cumplimiento

`SandboxMode` rige únicamente los efectos de archivo. `read-only` pide al backend que deniegue las escrituras — los runners POSIX conceden además el sumidero `/dev/null` que requieren sus shells, mientras que el runner de ACL de Windows no concede ninguna raíz explícitamente escribible e informa de cumplimiento parcial por sus huecos ambientales de ACL; `workspace-write` permite escrituras bajo la raíz del workspace y el área temporal prometida por el backend; `danger-full-access` se salta el confinamiento. La visibilidad de red y de procesos queda fuera de este vocabulario.

```ts type-equiv
/**
 * File-effect policy for confined processes. `read-only` permits only required
 * sinks such as `/dev/null`; `workspace-write` also permits the workspace and a
 * backend-defined temp area; `danger-full-access` bypasses confinement. Network
 * and process visibility are outside this vocabulary.
 */
type SandboxMode = 'read-only' | 'workspace-write' | 'danger-full-access'
```

Solo los dos primeros modos pueden enviarse a un provider. Un Consumer `danger-full-access` lanza su argv original y no llama a `ctx.sandbox`.

```ts type-equiv
/** A confining (non-`danger-full-access`) mode — the modes a {@link SandboxPolicy} can carry. */
type ConfinedSandboxMode = Exclude<SandboxMode, 'danger-full-access'>
```

El cumplimiento es un hecho que se informa. `full` significa que el backend rige cada efecto de archivo prometido por el modo; `partial` significa que un backend activo o una ABI de kernel más antigua rige solo un subconjunto, así que los Consumer que requieren la promesa absoluta deben rechazar o exponer esa distinción. Las ABI de Landlock antiguas y los límites de Everyone/enlaces duros del runner de ACL de Windows son los casos parciales actuales.

```ts type-equiv
/**
 * Enforcement completeness for this host. `partial` means an active backend or
 * older kernel ABI cannot govern every promised file effect; callers requiring
 * an absolute boundary must not treat it as `full`.
 */
type SandboxEnforcement = 'full' | 'partial'
```

## Política por llamada

La política de ejecución completa se resuelve y se transporta en cada llamada de capacidad. Incluye `danger-full-access` para que un Consumer pueda resolver la política una sola vez antes de decidir si se salta el confinamiento. Las llamadas de herramienta normales derivan `workspaceRoot` del cwd inmutable de la sesión que llama; la configuración de despliegue es el fallback sin agent. La raíz se canoniza con semántica de sistema de archivos antes de la normalización léxica, así que un cwd que contenga `symlink/..` identifica el directorio donde realmente se ejecuta el proceso lanzado.

```ts type-equiv
/**
 * The complete file-effect policy resolved for one capability call. The root
 * is carried even under modes that do not consume it so callers can resolve
 * policy once before choosing the enforcement path.
 */
interface SandboxExecutionPolicy {
  /** The file-effect mode this execution runs under. */
  mode: SandboxMode
  /** Absolute root directory `workspace-write` may write under. */
  workspaceRoot: string
  /**
   * Opaque identity of the calling session (the branded `dsh-session`
   * SessionId). Backends key per-session state off it (e.g. windows-acl gives
   * each live session/workspace pair a random private temp directory and SID,
   * while the workspace SID and standing grant remain per-workspace); absent
   * for agentless calls, which fall back to per-call backend state.
   */
  sessionId?: SessionId
}
```

`ctx.sandboxPolicy.resolve()` acepta la sesión activa y, para un reintento aprobado, un modo explícito. El servicio es el dueño de la precedencia y del fallback de raíz para que bash y fs no lo repitan.

```ts type-equiv
/** Inputs that select the sandbox policy for one capability call. */
interface SandboxPolicyRequest {
  /** Calling session; its immutable cwd becomes the workspace boundary. */
  session?: Session
  /** Explicit approved mode override, which outranks session policy. */
  mode?: SandboxMode
}
```

Solo una ejecución confinada llega a `ctx.sandbox`; la política de su provider estrecha el modo conservando la misma raíz. Esto permite que sesiones concurrentes, Consumer y reintentos puntuales escalados pidan al mismo provider límites distintos sin mutar el estado del provider.

```ts type-equiv
/**
 * What one confined execution is allowed to touch — carried PER CALL, not
 * fixed on the provider: two consumers may confine under different policies
 * at the same instant (bash under `read-only` while a confined child agent
 * needs its state directory writable), and an approved escalated retry is a
 * new call with a wider policy. Defaulting/resolution is an explicit step at
 * the consumer boundary; the provider treats the policy as fully specified.
 */
interface SandboxPolicy extends SandboxExecutionPolicy {
  /** The file-effect mode this execution runs under. */
  mode: ConfinedSandboxMode
}
```

## El argv envuelto y los dialectos de clasificación

`RunnerFailureRule` combina la evidencia de que un runner falló antes de ejecutar el comando. Un Consumer exige una salida distinta de cero, la compuerta opcional de códigos de salida permitidos y una firma fatal insensible a mayúsculas dentro de una línea restante de stderr. Primero se eliminan las exclusiones informativas por igualdad exacta de línea completa e insensibles a mayúsculas, así que un aviso benigno del runner no puede probar el fallo por sí solo. La línea coincidente permanece disponible como detalle del error; la clasificación no reescribe el stderr.

```ts type-equiv
/**
 * Evidence that identifies a sandbox runner failing before it executes the
 * wrapped command. A consumer first applies {@link allowedExitCodes} when
 * present, removes {@link informationalLines} by case-insensitive exact line
 * equality, then matches {@link fatalSignatures} case-insensitively within
 * each remaining stderr line. Exit status alone never proves runner failure.
 */
interface RunnerFailureRule {
  /** Nonzero process exit codes on which this rule may match; omitted permits any nonzero exit. */
  allowedExitCodes?: readonly number[]
  /** Non-empty substrings identifying a fatal runner diagnostic on one stderr line. */
  fatalSignatures: readonly string[]
  /** Benign stderr lines excluded by exact full-line equality before fatal matching. */
  informationalLines?: readonly string[]
}
```

`ConfinedArgv` es lo que lanza el Consumer. Además del argv de reemplazo, transporta el hecho de cumplimiento del backend y dos clasificadores ortogonales de stderr. `denialSignatures` identifica el comando confinado que está siendo bloqueado mientras el sandbox funciona correctamente. `runnerFailureRules` identifica al runner del sandbox que se niega o falla antes de ejecutar el comando; los Consumer los comprueban primero y exponen un fallo de infraestructura del sandbox, nunca un fallo de tarea ordinario.

```ts type-equiv
/**
 * A {@link SandboxProvider.confine} result: the argv to spawn in place of
 * the caller's own, plus the enforcement completeness the selected backend
 * achieves for it.
 */
interface ConfinedArgv {
  /** The wrapped argv (runner, profile, separator, then the caller's argv). */
  argv: string[]
  /** How completely the selected backend enforces the policy's file effects. */
  enforcement: SandboxEnforcement
  /**
   * The selected backend's denial DIALECT: the case-insensitive stderr
   * substrings a file effect denied by THIS backend produces (EROFS text
   * under bwrap's read-only binds, EACCES under Landlock, EPERM under
   * Seatbelt). A consumer that infers denials from a failed run's stderr
   * matches against exactly these rather than a cross-backend union — the
   * union claims denials a given backend never produces.
   */
  denialSignatures: readonly string[]
  /**
   * Structured runner-failure evidence rules. Consumers require a matching
   * fatal stderr line (after informational exclusions) and any rule-specific
   * exit-code gate before checking denial signatures: runner failure means the
   * command never ran, while denial means confinement worked and blocked it.
   */
  runnerFailureRules: readonly RunnerFailureRule[]
}
```

El [provider local](../../packages/sandbox/sandbox-local/README.es.md) es el dueño de la configuración del operador y traduce el dialecto de su runner a estas reglas. El [Consumer bash en sandbox](../../packages/shell/bash-sandbox/README.es.md) es el dueño del spawn y de la atribución de resultados.

## El provider y los errores de fallo cerrado

`ctx.sandbox.confine(argv, policy)` devuelve un `ConfinedArgv` o lanza `SandboxUnavailableError` con el código `SANDBOX_UNAVAILABLE` cuando no existe ningún backend utilizable. Los Consumer también pueden clasificar un fallo al lanzar u observar el argv devuelto; esa atribución pertenece al contrato del Consumer. El paso directo silencioso sin confinar nunca es legal para una política confinada.

La selección del provider, el sondeo, el caché y los informes de cumplimiento específicos de cada backend pertenecen al [provider local](../../packages/sandbox/sandbox-local/README.es.md).

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## API de Cordis

Generado desde la fuente por `scripts/gen-cordis-catalog.ts` (verificado como fresco por `pnpm run verify-cordis-catalog` en doc-sync; regenéralo con `pnpm run gen-cordis-catalog`) — los lados de idioma solo difieren en las rutas de los documentos emparejados específicas de cada localización. Los bloques de firmas usan un recinto `ts cordis-catalog` y conservan el JSDoc original de la fuente; los modos de despacho están definidos en el [primer](../cordis-primer.es.md#dispatch-modes), y la API `ctx` heredada del framework vive en [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxsandbox--sandboxprovider-abstract-seam"></a>

### `ctx.sandbox` — `SandboxProvider` (seam abstracto)

Servicio abstracto de sandbox de procesos. confine debe devolver un argv que confine o fallar cerrado en el momento del envoltorio o de la ejecución del runner; el paso directo silencioso sin confinar está prohibido. Las sondas funcionales arbitran las cadenas de varios runners y pueden omitirse para un candidato único, cuya propia negativa sigue siendo el extremo de fallo cerrado.

```ts cordis-catalog
/**
 * Wrap `argv` so it executes confined under `policy` on this host; the
 * caller spawns the returned argv in place of its own.
 * @param argv - the exact argv the caller is about to spawn (program plus
 *   arguments), NOT a shell string — a shell-shaped consumer passes
 *   `['bash', '-c', command]`.
 * @param policy - the file-effect policy this execution runs under,
 *   carried per call (see {@link SandboxPolicy}).
 * @returns the argv to spawn instead, plus the enforcement completeness
 *   the selected backend achieves for it.
 */
abstract confine(argv: readonly string[], policy: SandboxPolicy): ConfinedArgv
```

Source: [`packages/sandbox/sandbox/src/index.ts`](../../packages/sandbox/sandbox/src/index.ts)

<a id="ctxsandboxpolicy--sandboxpolicyservice"></a>

### `ctx.sandboxPolicy` — `SandboxPolicyService`

El servicio de política de sandbox (`ctx.sandboxPolicy`). Es el dueño del modo por defecto del despliegue, de la raíz de workspace de fallback y de la sección de política actual en tiempo de solicitud. Las capas de herramientas llaman a resolve en cada ejecución para que el log de modo de una sesión y su cwd inmutable viajen juntos hasta cada capacidad que confina.

```ts cordis-catalog
/**
 * Resolve the complete policy for one capability call. An approved explicit
 * mode outranks the session's last `sandbox/mode` event, which outranks the
 * deployment default. A session cwd is its workspace-write boundary; the
 * configured root is the fallback for agentless calls and sessions without a
 * cwd.
 * @param request - optional session and approved mode override.
 * @returns the fully resolved per-call mode and absolute workspace root.
 */
resolve(request: SandboxPolicyRequest = {}): SandboxExecutionPolicy

/**
 * Read the session override without applying the deployment default.
 * @param session - session whose log supplies the override.
 * @returns the last logged mode, or `undefined` without one.
 */
overrideOf(session: Session): SandboxMode | undefined
```

Types: [Session](session.es.md)

Source: [`packages/sandbox/sandbox-policy/src/index.ts`](../../packages/sandbox/sandbox-policy/src/index.ts)
<!-- END GENERATED cordis-surface -->
