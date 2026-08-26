# Ejecutor bash

[English](shell.md) | Español

El seam de ejecución bash se reparte entre una Service Definition ([dsh-shell](../../packages/shell/shell), `ctx.shell`), Service Providers ([dsh-bash-local](../../packages/shell/bash-local) y [dsh-bash-sandbox](../../packages/shell/bash-sandbox)) y un Consumer ([dsh-tool-bash](../../packages/shell/tool-bash), el schema `bash`). Los ids, la propiedad y los controles de los jobs genéricos en segundo plano viven en [jobs.md](jobs.es.md); este seam devuelve un manejador de proceso sin noción de tarea. La mecánica cruda de los grupos de procesos vive detrás del [seam de subprocesos](subprocess.es.md).

Fuente: [`packages/shell/shell/src/types.ts`](../../packages/shell/shell/src/types.ts)

## Namespace del entorno de shell gestionado

Las variables `DSH_*` son hechos de procesos hijos propiedad del harness. La herramienta bash orientada al modelo las recoge a través de `ctx.shellEnv` y las pasa mediante `ShellExecRequest.dshEnv`; el servicio de subprocesos elimina los nombres `DSH_*` heredados antes de fusionar la instantánea actual. El vocabulario `DshEnvironmentKey`/`DshEnvironment` pertenece al [seam de subprocesos](subprocess.es.md) y `dsh-shell` lo reexporta.

## Petición frente a spec: la división de `resolve()`

El seam separa la **petición orientada al modelo/plugin** (`workdir`/`timeoutMs`/`stdoutMaxBytes` opcionales, rellenados desde la configuración o la política de la petición) de la **spec totalmente resuelta** sobre la que actúa el ejecutor (con esos campos obligatorios). La capa de herramientas llama a `ctx.shell.resolve(request)` entre ambos (la regla del repo «explícito > implícito en los límites de paquete»); una `ShellExecSpec` transporta los valores resueltos.

```ts type-equiv
/**
 * A caller's execution REQUEST: `workdir` and `timeoutMs` are optional and
 * filled by {@link ShellExecutor.resolve} from the implementation's config.
 * This is the model-/plugin-facing shape; pass it to `resolve()` to obtain a
 * fully-resolved {@link ShellExecSpec}.
 */
interface ShellExecRequest {
  command: string
  /** Working directory override (default: implementation-configured). */
  workdir?: string | undefined
  /** Timeout override in milliseconds (implementations cap it). */
  timeoutMs?: number | undefined
  /**
   * Foreground stdout capture budget in bytes. Absent uses the executor's
   * default output cap. Trusted in-process consumers use this when they must
   * parse complete stdout up to their own bounded limit; the model-facing bash
   * tool does not expose it as a parameter.
   */
  stdoutMaxBytes?: number | undefined
  /** Abort signal — implementations kill the command when it fires. */
  signal?: AbortSignal | undefined
  /**
   * Bytes to write to the command's stdin, then close it. Absent leaves stdin
   * closed/empty (the default for model-driven tool calls). Set by in-process
   * plugins (e.g. the hooks bridges, which write a hook command's JSON payload
   * to its stdin); the model-facing bash tool does not expose it as a parameter
   * (a model that needs stdin uses shell syntax like a heredoc or a pipe).
   */
  stdin?: string | undefined
  /**
   * Ordinary environment entries for the command, merged after the credential
   * scrub. Managed facts belong in {@link dshEnv}, which merges after this
   * map, so an entry here can never displace one. Set by in-process plugins
   * (the hooks bridges set `CLAUDE_PROJECT_DIR`, `CLAUDE_PLUGIN_ROOT`, …); the
   * model-facing bash tool does not expose it as a parameter.
   */
  env?: Record<string, string> | undefined
  /**
   * Harness-owned `DSH_*` variables for this execution (typed to managed
   * keys). Executors discard ambient `DSH_*` entries before merging this
   * snapshot last, so an unavailable current fact cannot inherit a stale
   * value from the harness process and a caller {@link env} entry cannot
   * displace a managed one.
   */
  dshEnv?: DshEnvironment | undefined
  /** Fully resolved per-call sandbox policy; sandboxing executors default it. */
  sandboxPolicy?: SandboxExecutionPolicy | undefined
}
```

```ts type-equiv
/**
 * A resolved execution spec. {@link ShellExecutor.resolve} fills and caps the
 * required fields; {@link ShellExecutor.start} ignores `timeoutMs` because
 * background processes have no executor timeout.
 */
interface ShellExecSpec {
  command: string
  workdir: string
  timeoutMs: number
  /**
   * Resolved foreground stdout capture budget in bytes. `run()` uses it for
   * stdout; background jobs and stderr keep the executor's own output cap.
   */
  stdoutMaxBytes: number
  /** Abort signal — implementations kill the command when it fires. */
  signal?: AbortSignal | undefined
  /** Bytes to write to stdin before closing it; absent means no stdin. */
  stdin?: string | undefined
  /**
   * Ordinary environment entries carried through from
   * {@link ShellExecRequest.env}; {@link dshEnv} still merges after them.
   * OPTIONAL on the spec for the same reason as `stdin`: absent means no
   * ordinary extra environment.
   */
  env?: Record<string, string> | undefined
  /** Managed `DSH_*` snapshot (typed to managed keys); merges after {@link env}. */
  dshEnv?: DshEnvironment | undefined
  /** Resolved sandbox policy; ignored by executors that do not confine. */
  sandboxPolicy: SandboxExecutionPolicy | undefined
}
```

`stdin` y `env` son entradas de plugin de confianza en el mismo proceso y `dsh-tool-bash` no las expone. El ejecutor local limpia las credenciales ambientales antes de fusionar el env explícito aportado por el llamador. Consulta [la Agent Note bash-stdin-env](../../.agents/notes/implemented/architecture/2026-06-30-bash-stdin-env-trusted-plugin-api.es.md).

`stdoutMaxBytes` también es solo para plugins de confianza. Permite que un Consumer en primer plano solicite stdout completo hasta un presupuesto de análisis acotado sin alterar stderr, los jobs en segundo plano ni el tope de salida habitual de la herramienta bash orientada al modelo.

## Ejecuciones en primer plano: `ShellRunResult`

El resultado de una ejecución en primer plano completada (o terminada). Los resultados ortogonales se informan **de forma independiente** — un proceso puede a la vez agotar su timeout Y salir con 0 porque capturó la señal — así que `timedOut`, `aborted`, `signal` y `exitCode` son cada uno su propio campo; un llamador nunca interpreta una ejecución interrumpida como un éxito limpio.

```ts type-equiv
/** The outcome of one completed (or killed) foreground run. */
interface ShellRunResult {
  /** Exit code; null when the process died from a signal. */
  exitCode: number | null
  /** Terminating signal (e.g. 'SIGTERM'); null on normal exit. */
  signal: NodeJS.Signals | null
  /**
   * True when the executor's own timeout was the FIRST cause to cut the command
   * short. Mutually exclusive with {@link aborted}: one fused deadline drives
   * both the timeout and the caller's cancellation, so a timeout and an abort
   * racing before process close report the single first-abort cause, not both
   * (see the [timeout-library Agent Note](../../../../.agents/notes/implemented/architecture/2026-07-06-timeout-deadline-library.md)).
   */
  timedOut: boolean
  /**
   * True when the caller's `AbortSignal` was the FIRST cause to kill the command
   * (and it was not the executor's own timeout). Mutually exclusive with
   * {@link timedOut} — see there for the first-cause classification.
   */
  aborted: boolean
  /** The effective timeout applied to this run (after defaulting/capping). */
  timeoutMs: number
  stdout: CollectedOutput
  stderr: CollectedOutput
  /** Sandbox execution facts, absent for an unsandboxed executor. */
  sandbox?: ShellSandboxInfo
}
```

Cada flujo es un `CollectedOutput` — el texto (posiblemente truncado) más la información de recuperación; al truncarse, `text` es el **tramo final** y el flujo completo se vuelca a un archivo privado. Los campos pertenecen al [seam de subprocesos](subprocess.es.md) y `dsh-shell` los reexporta.

## Sandbox de archivos: `ShellSandboxInfo`

Un ejecutor que consume sandbox expone su modo de respaldo configurado a través de `ShellExecutor.sandboxMode`. La capa de herramientas pide a [`@deepseek-ai/dsh-sandbox-policy`](../../packages/sandbox/sandbox-policy/README.es.md) que resuelva la sobrescritura persistente `sandbox/mode` de cada sesión llamante y el cwd inmutable en `ShellExecRequest.sandboxPolicy`; una llamada estrictamente más amplia aprobada por el usuario solo reemplaza el modo. El vocabulario mode/root/enforcement pertenece al [seam `@deepseek-ai/dsh-sandbox`](sandbox.es.md); los modos solo rigen los efectos sobre archivos.

Una ejecución con sandbox informa de su modo, la clasificación conservadora de denegaciones y la completitud del enforcement. `runnerFailed` marca un fallo del runner de sandbox antes de que el comando llegara a ejecutarse; la ejecución en primer plano lanza `SANDBOX_UNAVAILABLE`, mientras que un proceso en segundo plano ya resuelto solo tiene su canal de hechos.

```ts type-equiv
/**
 * Sandbox facts for one run, present iff a sandboxing executor handled it.
 * Facts are reported independently of process exit status so callers can
 * distinguish command failures from policy denials and runner failures.
 */
interface ShellSandboxInfo {
  /** The mode the command actually ran under. */
  mode: SandboxMode
  /** Whether the sandbox denied a file operation. */
  denied: boolean
  /** How completely the selected runner enforced the requested mode. */
  enforcement?: SandboxEnforcement
  /** Whether the sandbox runner failed before the command could run. */
  runnerFailed?: boolean
}
```

El código de error `SANDBOX_UNAVAILABLE` (propiedad del [seam de sandbox](sandbox.es.md)) es lo que lanza el provider `ctx.sandbox` — y propaga el ejecutor — cuando un modo confinado no tiene backend utilizable. Un runner seleccionado que rechaza su profile llega al mismo error fail-closed en primer plano; un job en segundo plano ya resuelto registra `runnerFailed`. El modelo recibe los hechos de denegación/runner en los resultados, solo conoce el modo efectivo cuando un marcador de denegación lo nombra y puede solicitar un reintento único estrictamente más amplio mediante `sandbox_permissions` más `justification`; `ctx.approval` debe conceder esa llamada exacta antes de que se ejecute nada. El diseño completo de la política y del cambio de modo es la [Agent Note de sandbox](../../.agents/notes/implemented/feature/2026-07-06-sandbox.es.md).

## Procesos en segundo plano: `ShellProcess`

`start()` devuelve un manejador sin id ni propietario. `dsh-tool-bash` lo adapta a los hooks de `ctx.jobs.start()`; a partir de ahí, el runtime genérico es dueño de la identidad y el ciclo de vida de los jobs. `done` se resuelve cuando el proceso se cierra y nunca se rechaza, las lecturas siguen siendo válidas tras la resolución y los hechos de sandbox se fijan antes de que `done` se resuelva.

```ts type-equiv
/**
 * A background process handle returned by {@link ShellExecutor.start}. It is the
 * only access path; buffered output remains readable after exit. Composition
 * teardown (the subprocess service's disposal) kills running processes and
 * awaits {@link done}; an executor-only reload leaves them running.
 */
interface ShellProcess {
  /** Process lifecycle state (settled exactly once). */
  status: ShellProcessStatus
  /** Exit code once finished (null = killed by signal / still running). */
  exitCode: number | null
  /** Terminating signal name, when signal-killed. */
  signal: NodeJS.Signals | null
  /** Resolves when the underlying process closes (never rejects — a spawn failure settles as `killed` with the error on stderr). */
  readonly done: Promise<void>
  /** Sandbox facts, stamped once a confined process settles. */
  sandbox?: ShellSandboxInfo
  /**
   * Read output produced since the previous read (consuming — consecutive
   * reads never re-deliver). Reads that lost data flag `lossy` and point at
   * full-stream spill files when available.
   */
  readOutput(): ShellProcessRead
  /**
   * Kill the process group. Returns false when it had already finished
   * (no-op); idempotent.
   */
  kill(): boolean
}
```

`readOutput()` devuelve el delta incremental y los hechos de recuperación del spill:

```ts type-equiv
/** One incremental {@link ShellProcess.readOutput} read. */
interface ShellProcessRead {
  /** Output produced since the previous read (stderr in a marked section). */
  delta: string
  /** True when truncation dropped unread bytes the delta cannot include. */
  lossy: boolean
  /** Full stdout spill file, when stdout truncation occurred and a safe path is available. */
  stdoutSpillPath?: string
  /** Full stderr spill file, when stderr truncation occurred and a safe path is available. */
  stderrSpillPath?: string
}
```

## El servicio

`ShellExecutor` es dueño de `resolve`, del `run` en primer plano, del `start` de procesos en segundo plano y del hecho de capacidad `sandboxMode`. `dsh-bash-local` es dueño de los valores por defecto de los comandos, la clasificación de timeout/abort, el entorno de terminal y la fusión de lecturas en segundo plano; los grupos de procesos, los recolectores acotados, los archivos spill, la limpieza de credenciales y la quiescencia del disposal son cosa del [servicio de subprocesos](subprocess.es.md). `dsh-tool-bash` es dueño de la renderización orientada al modelo y adapta los manejadores en segundo plano al [runtime de jobs genérico](jobs.es.md). `dsh-shell` es dueño del contrato compartido de estado de salida de las herramientas de shell: el `parseExitStatus`/`ParsedExitStatus` exportado invierte los marcadores `[exit code: N]` / `[killed by signal: X]` que añaden el `renderResult` de `dsh-tool-bash` y el `renderPwshResult` de `dsh-tool-pwsh`, y el `presentResult` de ambas herramientas lo usa para dividir el texto renderizado en el cuerpo de salida de la tarjeta del terminal y su píldora de estado de salida.

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxshell--shellexecutor-abstract-seam"></a>

### `ctx.shell` — `ShellExecutor` (abstract seam)

Abstract bash execution service. Subclass, implement the abstract methods, and load the subclass as a plugin — it registers as `ctx.shell` (one implementation per context; loading a second throws, which is cordis' standard duplicate-service behavior).

Implementations must honor these semantics:

- run rejects only for infrastructure failures. Nonzero exits, timeout kills, and abort kills resolve with a ShellRunResult.
- start returns immediately; no timeout applies to background processes. `done` settles at process close and never rejects; spawn failures settle as `killed` with the error on stderr.
- ShellProcess.readOutput is incremental: consecutive reads never repeat output. Lossy reads report truncation and available spill files.
- A still-running background process is stopped and awaited when its owning composition tears down. With the subprocess seam that boundary is `ctx.subprocess` disposal, so a background process survives an executor-only reload.

```ts cordis-catalog
/**
 * Apply implementation-owned defaults and caps to a request before execution.
 * @param request - the caller's request; omitted fields get this
 *   implementation's defaults, capped fields are clamped.
 * @returns the fully-specified spec to hand to {@link run}/{@link start}.
 */
abstract resolve(request: ShellExecRequest): ShellExecSpec

/**
 * Run a command in the foreground; resolves when it finishes.
 * @param spec - a resolved spec from {@link resolve}, never a raw request.
 * @returns the outcome; nonzero exits, timeout kills, and abort kills
 *   resolve with a descriptive result rather than reject.
 */
abstract run(spec: ShellExecSpec): Promise<ShellRunResult>

/**
 * Start a background process and return its handle immediately.
 * @param spec - a resolved spec from {@link resolve}, never a raw request.
 * @returns the live process handle (reads, kill, quiescence promise).
 */
abstract start(spec: ShellExecSpec): ShellProcess
```

Source: [`packages/shell/shell/src/index.ts`](../../packages/shell/shell/src/index.ts)

<a id="ctxshellenv--shellenvregistry"></a>

### `ctx.shellEnv` — `ShellEnvRegistry`

Registry (`ctx.shellEnv`) for trusted, per-execution `DSH_*` variables. The namespace is rebuilt for every model shell call: ambient `DSH_*` values are discarded by the executor, then the registry's current snapshot is injected. Built-in shell facts remain owned by the registry itself while plugins can register additional, enumerable facts with effect-scoped disposal.

```ts cordis-catalog
/**
 * Register one environment contributor. Names and keys are unique; built-in
 * keys are reserved. Registration is disposed with the calling plugin fiber.
 * @param contributor - declared key ownership and per-execution resolver.
 * @returns the disposer that unregisters the contribution.
 */
register(contributor: BashEnvContributor): () => void

/**
 * Build the trusted `DSH_*` snapshot for one shell tool execution.
 * @param execution - the current tool execution.
 * @returns an immutable environment overlay containing built-ins and current contributions.
 */
collect(execution: ToolExecution): DshEnvironment

/**
 * Enumerate plugin-contributed variables without executing their resolvers.
 * @returns declarations sorted by environment variable name.
 */
list(): BashEnvVariableInfo[]
```

Types: [DshEnvironment](subprocess.es.md) · [ToolExecution](tools.es.md)

Source: [`packages/shell/shell-env/src/index.ts`](../../packages/shell/shell-env/src/index.ts)
<!-- END GENERATED cordis-surface -->
