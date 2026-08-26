# Agent Note: Una primitiva compartida de timeout/plazo, con la terminación forzosa dejada a cada capacidad

Status: implemented

[English](2026-07-06-timeout-deadline-library.md) | Español

## Problema

El manejo de timeouts se estaba desincronizando entre las capacidades con herramientas, y la divergencia no era superficial — era la misma lógica reimplementada de tres maneras, cada una con su propia carga sutil de corrección.

- **bash** (entonces en el `run.ts` de la implementación bash-local) tenía un timeout completo y correcto dentro de la plomería de procesos: un `timeoutMs` limitado por config, dos disparadores independientes — un `killTimer` para el timeout y un listener `onAbort` para la cancelación upstream —, cada uno llamando a un cierre `kill()` que escala SIGTERM→grace→SIGKILL sobre el grupo de procesos, y dos booleanos de resultado ortogonales (`timedOut`, `aborted`) enclavados de forma independiente. Tras esta consolidación, la plomería — hoy [packages/subprocess/subprocess-local/src/spawn.ts](../../../../packages/subprocess/subprocess-local/src/spawn.ts) — solo reacciona a los aborts; [packages/shell/bash-local/src/index.ts](../../../../packages/shell/bash-local/src/index.ts) es dueño del plazo fusionado y de la clasificación `timedOut`/`aborted`.
- **web_fetch** ([packages/web/web-fetch-http/src/provider.ts](../../../../packages/web/web-fetch-http/src/provider.ts)) tenía un timeout correcto pero *hecho a mano*: construía un `AbortController`, cableaba `setTimeout(() => controller.abort(new WebError(…, 'WEB_FETCH_TIMEOUT')))`, añadía y quitaba manualmente el listener de la señal upstream, limpiaba el temporizador en un `finally` y recuperaba la razón del timeout de `signal.reason` en un helper `translateAbortOrNetwork` porque el reader expone un `AbortError` desnudo.
- **web_search** ([packages/web/tool-web/src/search.ts](../../../../packages/web/tool-web/src/search.ts)) no tenía **ningún timeout**: `WebSearchRequest` ([packages/web/web/src/types.ts](../../../../packages/web/web/src/types.ts)) no lleva ningún campo `timeoutMs`, y el `search()` de cada provider solo reenvía `exec.signal`. (web_search sigue sin tiempo aquí — véase Consecuencias.)

Cada herramienta nueva de proceso externo o red volvía a derivar las mismas cuatro cosas — limitar el valor solicitado, arrancar un temporizador, fusionar el timeout con la cancelación upstream y distinguir «timed out» de «cancelled» a la salida — y la fusión y la recuperación de la razón son exactamente las partes fáciles de equivocar sutilmente (el baile de `signal.reason` de web_fetch es la prueba). Al mismo tiempo, la *terminación* que cada una realiza es irreductiblemente distinta: bash mata un grupo de procesos del sistema operativo (el trabajo corre en un proceso hijo, fuera de este runtime, alcanzable solo por señal), mientras que web aborta un `fetch` dentro del proceso (undici derriba el socket). No hay ningún mecanismo único que pueda detenerlas a todas.

## Decisión

`@deepseek-ai/dsh-timeout` vive bajo `packages/util/` (par de `dsh-brand`) y es dueño de la mitad de *temporización y clasificación* del timeout; la mitad de *terminación* — la terminación forzosa — queda en la implementación de cada capacidad. Es una biblioteca de funciones puras, **no** un servicio o plugin de cordis: no toma ningún `ctx`, no registra nada, no mantiene estado entre llamadas y no emite eventos. Deliberadamente no existe ningún «servicio de timeout» central que tuviera que saber cómo detener el trabajo de cada capacidad — ese conocimiento es exactamente lo que un microkernel mantiene fuera de las capas compartidas, y lo que demuestra el alcance exec-only `ExecExpiration` de Codex.

### La API de la biblioteca

Cuatro funciones, una interfaz de watchdog y un tipo de razón:

```ts ignore-check
/** The internal reason attached to a timeout abort, so consumers can classify it after the fact. */
export class TimeoutReason extends Error {
  override name = 'TimeoutReason'

  constructor(readonly code: string, readonly timeoutMs: number) {
    super(`${code} after ${timeoutMs}ms`)
  }
}

/** Validate/fill a caller's optional positive hint from the backend's default, then cap at its max. */
export function clampTimeout(
  requested: number | undefined,
  def: number,
  max: number,
  name = 'timeoutMs',
): number

/**
 * Build a deadline signal that aborts on upstream cancellation OR on timeout,
 * with the timeout carrying a `TimeoutReason`. `timeoutMs <= 0` means "no
 * timeout" (background jobs): forward only the upstream signal, arm no timer.
 * The returned object's `[Symbol.dispose]` clears the timer — `using` for a
 * scope-lifetime consumer, a manual call for an event-lifetime one.
 */
export function deadline(
  upstream: AbortSignal | undefined,
  timeoutMs: number,
  code: string,
): { signal: AbortSignal; [Symbol.dispose](): void }

/** A stable signal plus one-at-a-time, timer-guarded async-iterator demand. */
export interface IdleWatchdog {
  readonly signal: AbortSignal
  next<T>(iterator: AsyncIterator<T>): Promise<IteratorResult<T>>
  pulse(): void
  [Symbol.dispose](): void
}

/** Arm only while one iterator `next()` is outstanding; rearm on later demand or out-of-band activity. */
export function idleWatchdog(
  upstream: AbortSignal | undefined,
  timeoutMs: number,
  code: string,
): IdleWatchdog

/** Recover the TimeoutReason from an aborted signal (or error); `code` scopes the match to this deadline's timer. */
export function timeoutOf(x: AbortSignal | { reason?: unknown }, code?: string): TimeoutReason | undefined
```

`deadline` fusiona una señal upstream con un temporizador de un solo disparo a través de `AbortSignal.any`, añade un `TimeoutReason` tipado y expone una limpieza del temporizador desechable. Los timeouts no positivos son un centinela interno de sin-timeout para el trabajo en segundo plano propiedad del backend; las sugerencias externas pasan por `clampTimeout` y deben ser positivas y finitas. Sin temporizador ni señal upstream, la función devuelve una señal que nunca aborta con la misma forma de disposición. `idleWatchdog` exige en cambio un intervalo positivo finito, mantiene una única señal fusionada estable para todo el flujo y arma su temporizador solo mientras una llamada `next()` del iterador está pendiente; la resolución lo desarma, la demanda posterior lo rearma, y `pulse()` rearma esa misma demanda pendiente tras actividad de transporte fuera de banda. Un pulse fuera de una demanda pendiente o tras la disposición es un no-op; la demanda concurrente falla, y la disposición limpia el armado activo. Los providers traducen las razones de timeout en resultados específicos del seam. `timeoutOf(signal, code)` acota la clasificación para que un deadline exterior anidado se trate como cancelación upstream y no como el timeout de la capacidad interior.

### La división del trabajo

| Preocupación | Dueño |
|---|---|
| Validar la sugerencia de la petición y limitar el default/max | `dsh-timeout` (`clampTimeout`) — aritmética pura más el contrato compartido de petición positiva-finita |
| Armar un temporizador de un solo disparo, abortar en el plazo, llevar la razón, fusionar con la cancelación upstream | `dsh-timeout` (`deadline`) |
| Armar y rearmar solo alrededor de la demanda pendiente del iterador, incluida la actividad fuera de banda | `dsh-timeout` (`idleWatchdog`) |
| Limpiar el temporizador | `dsh-timeout` (`[Symbol.dispose]` en cualquiera de las dos primitivas) |
| Clasificar la primera razón de abort tras el abort | `dsh-timeout` (`timeoutOf`) |
| **Terminar de verdad el trabajo** | la implementación de la capacidad |
| Los *valores* default/max | la config de la capacidad |
| La cadena `code` del timeout | la capacidad (`WEB_FETCH_TIMEOUT` ≠ `BASH_TIMEOUT`) |

La señal solo *notifica*; la terminación es siempre trabajo del listener, y el listener difiere por capacidad. bash escribe su propio `addEventListener('abort', kill)` porque el proceso del sistema operativo vive fuera de este runtime y nada más lo matará; web entrega `d.signal` a `fetch` y undici derriba el socket. Por eso la lectura/escritura/edición de archivos no toma **ningún** `timeoutMs`: una syscall local es como mucho abortable con esfuerzo máximo, un timeout no podría forzar a `fsync`/`rename` a detenerse, y añadir uno sería un default implícito que viola lo explícito-sobre-lo-implícito. Ambos agents de referencia dejan la E/S de archivos sin tiempo por la misma razón.

### Cómo consume cada capacidad la biblioteca

- **web_fetch** — la herramienta sigue siendo valida-y-reenvía; el controlador hecho a mano + `setTimeout` + listener manual + `finally` + recuperación de `signal.reason` del provider se sustituye por un `deadline`/`timeoutOf` propiedad del provider. Una señal upstream ya abortada sigue lanzando `WEB_ABORTED` por adelantado; si no, `fetch` corre contra la `d.signal` fusionada, y `translateAbortOrNetwork` clasifica un error lanzado por la señal (`timeoutOf` → `WEB_FETCH_TIMEOUT`, si no abortado → `WEB_ABORTED`, si no red → `WEB_PROVIDER_ERROR`). El contrato público de códigos de error no cambia, y `TimeoutReason` nunca cruza el seam de web como error público.
- **bash** — `resolve()` limita la petición a un spec explícito. El `run()` en primer plano crea el deadline y pasa su señal a la ejecución del proceso, cuyo listener de abort existente realiza la matanza del grupo de procesos. El ejecutor clasifica el primer abort como timeout o cancelación. Los inicios en segundo plano siguen sin timeout y solo reenvían la cancelación upstream.
- **Adaptadores LLM (modelo de lenguaje de gran tamaño)** — `dsh-llm-deepseek` y `dsh-llm-pi-ai` envuelven la iteración real del transporte con `idleWatchdog`. El intervalo configurado de cinco minutos cubre solo la demanda pendiente del provider, no el tiempo que el consumidor downstream pasa entre chunks. El adaptador directo de DeepSeek también pulsa esa demanda pendiente cuando su parser SSE observa un comentario, sin emitir el comentario como `StreamChunk` ni escribirlo en el registro de sesión. El SDK de pi-ai no expone la actividad de comentarios a su adaptador, de modo que esa vía solo puede rearmarse cuando el SDK cede. La señal estable llega a `fetch` o al SDK durante toda la llamada, de modo que el timeout cierra la petición subyacente y se mapea a `TIMEOUT`, mientras que un abort del llamante anterior se mapea a `ABORTED`.

## Consecuencias

- El resultado de `runBash` ya no enclava `timedOut` y `aborted` de forma independiente; un timeout y un abort del usuario compitiendo antes del cierre del proceso ahora informan una única causa de primer abort en lugar de ambas verdaderas. La matanza uniforme SIGTERM→grace→SIGKILL no cambia, y el tipo de la Service Definition `ShellRunResult` conserva ambos booleanos (ahora mutuamente excluyentes), de modo que el renderizado de resultados de `dsh-tool-bash` queda intacto.
- `SpawnSpec.timeoutMs` y `SpawnOutcome.timedOut`/`aborted` se eliminaron en lugar de conservarse como vestigios siempre-cero/siempre-falso: con `runBash` sin temporizador propio y el ejecutor dueño de la clasificación, no se leían en ningún sitio. Un campo siempre-0 que nadie lee es peso muerto bajo la compuerta de cobertura por archivo.
- web_fetch se deshizo de su controlador/temporizador/listener/recuperación-de-razón a medida; el clasificador ahora se basa en la señal de deadline (`timeoutOf` + `aborted`) en lugar de en la forma del error lanzado, lo que es robusto tanto en la fase de petición con rechazo-por-razón como en la fase de lectura con `AbortError` desnudo.
- `AbortSignal.any` y `using`/`Symbol.dispose` entran en el repo por primera vez aquí (baseline Node ≥ 24, ya cumplida).
- Los flujos de modelo comparten ahora un contrato de temporizador rearmable sin convertir un intervalo de inactividad deslizante en un plazo de llamada total ni cobrar el tiempo de pensamiento del consumidor. Los adaptadores que pueden observar actividad de transporte fuera de banda pueden pulsar una demanda pendiente; la actividad suprimida permanece invisible para el watchdog. La primitiva sigue solo notificando; las pruebas de los adaptadores demuestran que sus transportes observan su señal estable y terminan.

Fuera de alcance, nombrado para marcar la frontera: `web_search` puede ganar un `timeout_ms` opcional orientado al modelo una vez que se planifique su cobertura de tool-schema/instantáneas; las herramientas de descubrimiento fs basadas en ripgrep ([búsqueda ripgrep empaquetada](2026-08-01-packaged-ripgrep-search.es.md)) consumen la misma forma de deadline propiedad del provider a través de `dsh-tool-call-timeout-policy` y `exec.signal`; un middleware del waterfall de `tools/execute` podría armar un deadline por defecto para cada llamada de herramienta impulsando `exec.signal` — eso sería un plugin que *consume* esta biblioteca y sigue solo notificando, quedando la terminación forzosa como trabajo de cada capacidad.

## Alternativas consideradas

**Un *plugin*/servicio `ctx.timeout` unificado.** Rechazada por razones de microkernel. Un servicio que pudiera detener el trabajo de cualquier herramienta tendría que entender el mecanismo de terminación de cada capacidad (SIGKILL de grupo de procesos, derribo de socket, comprobaciones de frontera de syscall) — el «el kernel sabe demasiado» que la arquitectura prohíbe. El `ExecExpiration` de Codex está acotado a la familia exec precisamente porque la matanza que impulsa (`killpg`) es específica de la familia de procesos; MCP y los flujos de modelo mantienen las suyas. No existe ninguna capa intermedia coherente que sea dueña de la terminación de todo, de modo que la pieza compartida solo puede ser la mitad pura de temporización/clasificación — una biblioteca, no un servicio.

**Timeout ad-hoc por herramienta, sin código compartido (el statu quo previo, y la elección de Claude Code).** Rechazada porque ya estaba produciendo divergencia y duplicando la carga de corrección: web_fetch reimplementaba a mano exactamente la lógica de controlador/razón que las futuras herramientas de red/procesos tendrían que volver a derivar cada una, y la fusión + la recuperación de `signal.reason` son las partes propensas a error. Claude Code tolera la duplicación completa; este repo tiene un único canal de abort compartido (`exec.signal` en cada `execute`) que hace que una primitiva compartida pequeña sea estrictamente más limpia, así que la relación coste/beneficio difiere.

**Un envoltorio `withTimeout(promise, ms)` en lugar de una fábrica de señales.** Rechazada porque competir una promesa contra un temporizador resuelve la promesa de la *llamada de herramienta* en el plazo sin detener el trabajo subyacente — el proceso hijo o el socket de fetch se quedan filtrados. Entregar una señal y exigir que la capacidad escuche es lo que fuerza a que exista una vía de terminación real. Esto refleja la regla defensiva «dispose debe alcanzar la quiescencia, no solo pedirla».

**Mantener separados los disparadores de timeout y cancelación de bash.** Rechazada porque una única señal de deadline elimina el temporizador a medida y estandariza la clasificación. Las carreras informan de qué abort llegó primero, mientras que la vía de terminación SIGTERM-a-SIGKILL existente permanece sin cambios.
