# Subproceso

[English](subprocess.md) | Español

El seam de subprocesos se reparte entre una Service Definition ([dsh-subprocess](../../packages/subprocess/subprocess), `ctx.subprocess`) y un Service Provider ([dsh-subprocess-local](../../packages/subprocess/subprocess-local)); sus Consumers son otros seams de capacidad y backends fuera de proceso: la [familia de ejecutores bash](shell.es.md) usa la salida por lotes recopilada, LSP usa tuberías de protocolo sin procesar, el backend PTY usa la primitiva de terminal y el backend de subagente (subagent) del ACP usa ndjson por tubería más el stderr heredado. Este seam posee el espacio de nombres de entorno gestionado `DSH_*`, la depuración compartida de credenciales (`scrubbedParentEnv`) y la forma `CollectedOutput`; [dsh-shell](../../packages/shell/shell) reexporta el vocabulario para que los consumidores bash mantengan una única raíz de importación.

Fuente: [`packages/subprocess/subprocess/src/types.ts`](../../packages/subprocess/subprocess/src/types.ts) y [`packages/subprocess/subprocess/src/index.ts`](../../packages/subprocess/subprocess/src/index.ts)

## Búsqueda de ejecutables

Los directorios de trabajo del spawn, las rutas de ejecutables, los procesos ordinarios y las sesiones de terminal de un provider habitan el mismo espacio de rutas y procesos que el provider de sistema de archivos montado. `resolveExecutable(command, env?, signal?)` verifica las rutas de ejecutables absolutas o resuelve los nombres simples a través del `PATH` depurado del provider más las anulaciones deliberadas.

## Espacio de nombres de entorno gestionado y salida capturada

Las variables `DSH_*` son datos de procesos hijo propiedad del harness; las implementaciones descartan los nombres `DSH_*` ambientales antes de que se fusionen las `env` explícitas del llamante, de modo que un dato actual solo llega como una entrada de cadena deliberada, mientras que una tombstone explícita de `undefined` elimina un valor ambiental ordinario. Cada flujo recopilado informa su estado de truncamiento y de recuperación de spill a través de `CollectedOutput`.

```ts type-equiv
/** One environment key inside the managed {@link DSH_ENV_PREFIX} namespace. */
type DshEnvironmentKey = `${typeof DSH_ENV_PREFIX}${string}`
```

```ts type-equiv
/** Trusted DeepSeek Harness variables for one child-process execution. */
type DshEnvironment = Readonly<Record<DshEnvironmentKey, string>>
```

```ts type-equiv
/** One captured stream: the (possibly truncated) text plus recovery info. */
interface CollectedOutput {
  /** Collected text — the TAIL of the stream when truncated. */
  text: string
  /** True when bytes were dropped from `text`. */
  truncated: boolean
  /** Path to a file holding the COMPLETE stream, when truncated and available. */
  spillPath?: string
}
```

## Disposiciones de stdio con forma de Node

La disposición de cada flujo es explícita y se elige por consumidor: tuberías sin procesar para el encuadre de protocolo (JSON-RPC de LSP, ndjson de ACP), inherit para los diagnósticos de paso directo y modo collect para la salida por lotes acotada — con el archivo de spill opcional, de modo que una cola de diagnóstico (el stderr de un language server) se almacena en búfer sin dejar archivos.

```ts type-equiv
/**
 * stdin disposition. `'ignore'` leaves fd 0 on `/dev/null`; `'pipe'` exposes
 * {@link SubprocessHandle.stdin} for the caller's ongoing protocol writes;
 * `{ data }` writes the bytes and closes (the batch shape).
 */
type SubprocessStdinMode = 'ignore' | 'pipe' | { readonly data: string }
```

```ts type-equiv
/**
 * Bounded in-memory collection for one output stream, with an optional
 * full-stream spill file. Omitting `spill` keeps only the in-memory tail —
 * the diagnostic-tail shape (a language server's stderr); including it makes
 * the complete stream recoverable up to its cap (the bash tool shape).
 */
interface SubprocessCollect {
  /** In-memory cap in bytes; overflow keeps the TAIL. */
  maxBytes: number
  /** Full-stream spill file; absent disables spilling entirely. */
  spill?: {
    /** Whole-stream byte cap; a larger stream discards its now-incomplete spill. */
    maxBytes: number
  }
}
```

```ts type-equiv
/**
 * stdout/stderr disposition. `'pipe'` exposes the raw `Readable` for the
 * caller's protocol decoding; `'inherit'` passes the parent's descriptor
 * through (child diagnostics land on the harness's own stream); a
 * {@link SubprocessCollect} object buffers boundedly with offset-based reads.
 */
type SubprocessOutputMode = 'pipe' | 'inherit' | SubprocessCollect
```

```ts type-equiv
/** Per-stream stdio dispositions, all explicit — this seam applies no defaults. */
interface SubprocessStdio {
  stdin: SubprocessStdinMode
  stdout: SubprocessOutputMode
  stderr: SubprocessOutputMode
}
```

## La especificación de spawn totalmente explícita

El seam no aplica ningún valor por defecto: cada disposición, límite y directorio es explícito en la spec, de modo que la configuración del propio llamante —no un valor por defecto oculto del servicio de subprocesos— los decide. `argv` nunca se interpreta como shell.

```ts type-equiv
/**
 * A fully-specified spawn request. This seam applies no defaults: every
 * disposition, limit, and directory is explicit, so the caller's own config —
 * not a hidden subprocess-service default — decides them (the `dsh-shell`
 * request/spec split is the owning template).
 */
interface SubprocessSpawnSpec {
  /** Executable and arguments; `argv[0]` is the program. Never shell-interpreted here. */
  argv: readonly string[]
  /** Working directory for the child. */
  cwd: string
  /** Per-stream stdio dispositions. */
  stdio: SubprocessStdio
  /**
   * Positive finite grace period in milliseconds, no greater than
   * `MAX_TIMER_DELAY_MS`, for the {@link SubprocessHandle.terminate} escalation
   * and for draining still-open collected pipes after the process exits (an
   * inherited descriptor held by a surviving descendant cannot hold the
   * outcome open indefinitely).
   */
  graceMs: number
  /**
   * Abort signal — starts the terminate escalation on the process tree when
   * it fires. The caller owns deadlines and cause classification; this seam
   * only reacts to the abort.
   */
  signal?: AbortSignal | undefined
  /**
   * Explicit environment entries merged onto the implementation's scrubbed
   * parent base (see `scrubbedParentEnv`), with no namespace validation. A
   * string is a deliberate caller opt-in, so a forwarded credential-shaped
   * entry or current `DSH_*` fact survives the scrub; `undefined` is a
   * tombstone that removes an ordinary ambient entry from the child.
   */
  env?: NodeJS.ProcessEnv | undefined
}
```

## Manejadores: flujos, lectores y terminación con ámbito de árbol

Un spawn devuelve un manejador vivo de inmediato. Los lectores del modo collect toman offsets de bytes de todo el flujo y nunca consumen, de modo que los lectores independientes no pueden robarse los deltas entre sí; los flujos por tubería pertenecen al llamante. La terminación tiene ámbito de árbol en todas las plataformas: `terminate()` —el único verbo de terminación— escala SIGTERM→gracia→SIGKILL, y `waitForExit()` observa todo el árbol — suficiente para que un consumidor construya su propia escalera de desmontaje (el `disposeAcpChild` del backend de ACP, que cierra stdin primero, es la plantilla).

```ts type-equiv
/**
 * A live child process rooted in its own process tree. Collected output
 * remains readable after exit; piped streams belong to the caller.
 *
 * Termination is tree-scoped everywhere: POSIX signals the detached process
 * group (falling back to the direct child when the group is gone), Windows
 * terminates the tree via `taskkill /T`, so helper processes cannot outlive
 * the handle unnoticed.
 */
interface SubprocessHandle {
  /** Process id (tree root); -1 when the spawn itself failed. */
  readonly pid: number
  /** The child's stdin, present iff spawned with `stdin: 'pipe'`. */
  readonly stdin: Writable | undefined
  /** The child's raw stdout, present iff spawned with `stdout: 'pipe'`. */
  readonly stdout: Readable | undefined
  /** The child's raw stderr, present iff spawned with `stderr: 'pipe'`. */
  readonly stderr: Readable | undefined
  /** Offset-based readers for collect-mode streams (also readable after exit). */
  readonly collected: SubprocessCollectedOutputs
  /** Resolves at process close with exit facts; rejects only for spawn-level failures. */
  readonly done: Promise<SubprocessOutcome>
  /**
   * Begin the SIGTERM → `graceMs` → SIGKILL escalation on the process tree
   * (Windows force-terminates immediately) — the seam's only termination
   * verb. Idempotent, a no-op once the tree is gone (the pid may be reused),
   * and also triggered by the spec's abort signal.
   */
  terminate(): void
  /**
   * Wait until the process tree has exited — the tree, not just the direct
   * child, so a still-running helper is observable before teardown returns.
   * @param signal - optional bound for the wait.
   * @returns `true` when the tree exited, `false` when the signal aborted first.
   */
  waitForExit(signal?: AbortSignal): Promise<boolean>
}
```

```ts type-equiv
/**
 * Cursor-free incremental access to one collected output stream. Offsets are
 * whole-stream byte coordinates owned by the caller, so independent readers
 * cannot consume one another's output; `readFrom(0)` after settlement is the
 * batch result (`lossy` then means the in-memory tail lost its head — the
 * {@link CollectedOutput.truncated} fact).
 */
interface SubprocessOutputReader {
  /**
   * Read everything captured since `fromByte`. When that offset has slid out
   * of the in-memory tail window the read is `lossy` — it returns the whole
   * retained tail and the gap is only recoverable from the spill file.
   * @param fromByte - whole-stream offset to resume from (a prior read's `nextOffset`; 0 for the first read).
   * @returns the delta text, the next offset, the `lossy` flag, and the spill path when one exists.
   */
  readFrom(fromByte: number): SubprocessOutputRead
}
```

```ts type-equiv
/** One incremental {@link SubprocessOutputReader.readFrom} read. */
interface SubprocessOutputRead {
  /** Stream text from the requested offset (the whole retained tail when lossy). */
  text: string
  /** Whole-stream offset to resume from on the next read. */
  nextOffset: number
  /** True when the requested offset slid out of the in-memory tail window. */
  lossy: boolean
  /** Path to the full-stream spill file, when one was created and remains intact. */
  spillPath?: string
}
```

```ts type-equiv
/** Offset-based readers for the streams spawned in collect mode. */
interface SubprocessCollectedOutputs {
  /** Present iff stdout is a {@link SubprocessCollect}. */
  readonly stdout?: SubprocessOutputReader
  /** Present iff stderr is a {@link SubprocessCollect}. */
  readonly stderr?: SubprocessOutputReader
}
```


## Los resultados solo transportan datos de salida

`done` informa el vocabulario del evento `close` de Node y ninguna clasificación de causas — el servicio mata al abortar pero nunca decide el porqué (el llamante lee la señal de plazo que posee, p. ej. la división `timedOut`/`aborted` del ejecutor bash). La salida recopilada permanece legible a través de `handle.collected` después de la resolución, de modo que los llamantes por lotes y por streaming comparten una única vía de acceso.

```ts type-equiv
/**
 * Exit facts of one closed process — Node's `close`-event vocabulary.
 * Deliberately carries NO timeout or cancellation classification (the caller
 * reads the signal it owns to classify causes) and NO output: collected
 * streams stay readable through {@link SubprocessHandle.collected} after
 * settlement, so batch and streaming callers share one access path.
 */
interface SubprocessOutcome {
  /** Exit code; null when the process died from a signal. */
  exitCode: number | null
  /** Terminating signal (e.g. 'SIGTERM'); null on normal exit. */
  signal: NodeJS.Signals | null
}
```

## Primitiva de proceso de terminal

`spawnTerminal(spec)` es la primitiva de proceso sin tubería. El provider asigna el terminal de control y posee el transporte de texto UTF-8, la inspección y señalización del grupo de procesos en primer plano y una operación esperada de TERM a KILL que alcanza la quiescencia para cada miembro de la sesión que el provider aún pueda observar; los providers documentan los límites de observabilidad específicos de la plataforma subyacente. El backend PTY sigue siendo responsable de la detección del prompt, la inferencia de preparación, el scrollback, la política de sandbox y la propiedad de la sesión persistente; el `spawn()` ordinario no puede reconstruir la semántica del terminal de control.

La spec de terminal especifica por completo argv, cwd, anulaciones de entorno, dimensiones, gracia de limpieza y cancelación opcional de la asignación. Su manejador expone `pid`, salida ordenada, `done`, `write`, `inspectForeground`, `signalForeground` y un `terminate` esperado; las formas públicas exactas se generan en el [catálogo del servicio `ctx.subprocess`](#ctxsubprocess--subprocessruntime-abstract-seam).

## Comportamiento del servicio

La Service Definition abstracta [`SubprocessRuntime`](../../packages/subprocess/subprocess/src/index.ts) especifica las coordenadas del mundo de ejecución, la búsqueda de ejecutables, el `spawn` ordinario y `spawnTerminal`. [`LocalSubprocessRuntime`](../../packages/subprocess/subprocess-local/src/index.ts) los proporciona con árboles de procesos separados, cableado por disposición, depuración de credenciales, `node-pty`, inspección de procesos de la plataforma y liberación con terminación y espera conjunta. Consulta [`dsh-subprocess`](../../packages/subprocess/subprocess/README.es.md) para el contrato de la Service Definition y [`dsh-subprocess-local`](../../packages/subprocess/subprocess-local/README.es.md) para la mecánica local.

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxe2b--e2bruntime"></a>

### `ctx.e2b` — `E2BRuntime`

Creates one lazily consumable E2B SDK handle and deletes the sandbox at timeout or disposal. Creation begins at plugin construction; adapters await getSandbox before their first operation.

```ts cordis-catalog
/**
 * Return the shared live SDK handle.
 * @returns the created sandbox after the configured cwd exists.
 * @throws when E2B rejects creation or the service is disposing.
 */
async getSandbox(): Promise<Sandbox>
```

Source: [`packages/e2b/e2b/src/index.ts`](../../packages/e2b/e2b/src/index.ts)

<a id="ctxsubprocess--subprocessruntime-abstract-seam"></a>

### `ctx.subprocess` — `SubprocessRuntime` (abstract seam)

Abstract subprocess service. Subclass, implement spawn, and load the subclass as a plugin — it registers as `ctx.subprocess` (one implementation per context; loading a second throws, which is cordis' standard duplicate-service behavior).

Implementations must honor these semantics:

- Executable paths belong to one execution world shared with the mounted filesystem provider.
- spawn returns immediately with a live handle; `done` resolves at process close with exit facts and rejects only for spawn-level failures.
- Collect-mode readers are offset-based and non-consuming, so independent readers never consume one another's output; lossy reads report truncation and the spill file holding the complete stream when one exists. Piped streams are handed to the caller raw and never buffered here.
- SubprocessHandle.terminate (and the spec's abort signal) escalates SIGTERM→grace→SIGKILL — the only termination verb — tree-scoped on every platform. SubprocessHandle.waitForExit observes whole-tree liveness, so a consumer-owned teardown ladder can hold each tier on real quiescence.
- Disposal of the service terminates all still-running managed processes and awaits their exit.
- spawnTerminal owns terminal allocation, text transport, foreground groups, signalling, and whole-session quiescence behind one awaited termination method; readiness and persistent-shell policy stay in the PTY consumer. Its output stream ends after queued terminal output when the top-level process exits.

```ts cordis-catalog
/**
 * Resolve one configured executable in this provider's execution world.
 * Absolute paths are verified; bare names use the provider's scrubbed PATH
 * plus explicit environment overrides. Relative paths containing separators
 * are rejected: the resolution base is undefined, so providers fail loud
 * instead of guessing.
 * @param command - absolute executable path or bare PATH name.
 * @param env - explicit environment entries used for lookup.
 * @param signal - aborts remote or local lookup.
 * @returns a canonical executable path.
 */
abstract resolveExecutable( command: string, env?: Readonly<Record<string, string>>, signal?: AbortSignal, ): Promise<string>

/**
 * Start one managed child process from a fully-specified spec; this seam
 * applies no defaults.
 * @param spec - argv, directory, stdio dispositions, grace, cancellation, and environment.
 * @returns the live process handle (streams/readers, signalling, outcome promise).
 */
abstract spawn(spec: SubprocessSpawnSpec): SubprocessHandle

/**
 * Allocate a real terminal and start one owned process session. This is the
 * only non-pipe process primitive: implementations own terminal byte I/O,
 * foreground groups, signals, and complete session-tree cleanup.
 * @param spec - fully specified argv, cwd, environment, dimensions, grace, and allocation cancellation.
 * @returns the live terminal handle after allocation succeeds.
 */
abstract spawnTerminal(spec: SubprocessTerminalSpawnSpec): Promise<SubprocessTerminalHandle>
```

Source: [`packages/subprocess/subprocess/src/index.ts`](../../packages/subprocess/subprocess/src/index.ts)
<!-- END GENERATED cordis-surface -->
