# dsh-timeout

[English](README.md) | Español

La mitad de **medición y clasificación** de un timeout — una biblioteca sin dependencias de funciones puras (sin dependencias de runtime del harness) compartida por toda capacidad que acota la pista de timeout del llamador, arma un plazo y luego debe distinguir «timed out» de «cancelled».

No es dueña de **ninguna terminación**. La señal que reparte solo *notifica*; detener realmente el trabajo permanece en cada capacidad, porque ese mecanismo difiere — bash hace SIGKILL a un grupo de procesos del SO, web derriba un socket `fetch` — y ninguna capa compartida puede ser dueña de todos ellos. Esta es la frontera que traza la [Agent Note](../../../.agents/notes/implemented/architecture/2026-07-06-timeout-deadline-library.md): comparte la medición/clasificación, mantén local la matanza dura.

Es una **biblioteca, no un servicio ni un plugin**: sin `ctx`, no registra nada, no mantiene estado, no emite eventos. Un «servicio de timeout» tendría que entender cómo detener el trabajo de todas las capacidades — exactamente el conocimiento que un microkernel mantiene fuera de las capas compartidas.

## API

```ts
import { clampTimeout, deadline, idleWatchdog, MAX_TIMER_DELAY_MS, timeoutOf, TimeoutReason } from '@deepseek-ai/dsh-timeout'
```

| Export | Rol |
|---|---|
| `clampTimeout(requested, def, max, name?)` | Valida la pista opcional del llamador (positiva y finita), rellena desde `def`, limita a `max`. Lanza (con `name`) ante una pista no positiva o no finita. |
| `deadline(upstream, timeoutMs, code)` | Funde la cancelación de `upstream` con un timeout en una sola `AbortSignal` (`AbortSignal.any`); el timeout lleva un `TimeoutReason`. `[Symbol.dispose]` limpia el temporizador. |
| `idleWatchdog(upstream, timeoutMs, code)` | Mantiene una señal fusionada estable y solo la arma mientras el `next()` del iterador asíncrono protegido está pendiente. La resolución desarma; la demanda posterior o la actividad de `pulse()` rearma; la disposición limpia; la demanda concurrente rechaza. |
| `MAX_TIMER_DELAY_MS` | El mayor retardo que Node programa sin acotarlo a un milisegundo (`2_147_483_647`). La configuración que posee temporizadores no debe superarlo. |
| `timeoutOf(signal \| { reason }, code?)` | Recupera el `TimeoutReason` de una señal/error anulado, si no `undefined` — el clasificador timeout-vs-cancel. Pasa `code` para comparar solo el temporizador de ESTE deadline (ver anidamiento más abajo). |
| `TimeoutReason` | La razón interna (`code` + `timeoutMs`) sellada en una anulación por timeout. No es un error público — los providers la traducen a su propio error/campo. |

## El centinela `timeoutMs <= 0`

`0` es el valor **interno** de «sin timeout» para el trabajo en segundo plano propiedad del backend (bash `start()`): `deadline()` no arma ningún temporizador y reenvía solo `upstream`; sin upstream tampoco, devuelve una señal que nunca anula más un disposer sin operación, de modo que todo llamador conserva una única forma de llamada. Las pistas externas de solicitud se validan como **positivas y finitas** mediante `clampTimeout` antes de llegar a `deadline`, por lo que `0` nunca es un valor de «desactivar timeout» visible para el modelo o los plugins.

## Forma de uso

```ts
import { deadline, timeoutOf } from '@deepseek-ai/dsh-timeout'

declare function runWork(options: { signal: AbortSignal }): Promise<unknown>

// Scope-lifetime consumer (foreground bash, one fetch): `using` disposes the timer.
export async function runWithDeadline(upstream: AbortSignal | undefined, timeoutMs: number): Promise<unknown> {
  using d = deadline(upstream, timeoutMs, 'BASH_TIMEOUT')
  const outcome = await runWork({ signal: d.signal })               // work listens on d.signal and terminates itself
  const timedOut = timeoutOf(d.signal, 'BASH_TIMEOUT') !== undefined // classify the first abort, scoped to OUR code
  const aborted = d.signal.aborted && !timedOut                     // mutually exclusive: timeout won, or cancel did
  return { outcome, timedOut, aborted }
}
```

La señal solo *notifica* — el llamador DEBE adjuntar su propia terminación (`d.signal.addEventListener('abort', kill)`, o entregar `d.signal` a `fetch`). Competir una promesa contra un temporizador resolvería la llamada de herramienta mientras el proceso hijo o el socket siguen filtrándose; repartir una señal obliga a que exista una vía real de terminación.

Pasa tu propio `code` a `timeoutOf` para que la clasificación se componga bajo anidamiento. Cuando `upstream` es a su vez una señal de deadline, `AbortSignal.any` preserva su `TimeoutReason` si ese temporizador se dispara primero. Acotar a tu código hace que un timeout ajeno se lea como una cancelación ordinaria de upstream en lugar de reclamar que expiró el temporizador local.

Para un transporte por flujo, crea un `idleWatchdog`, pasa su `signal` estable al transporte y llama a `watchdog.next(iterator)` por cada lectura del provider. Llama a `watchdog.pulse()` cuando la actividad del transporte no produzca un valor de iterador. El intervalo debe ser positivo, finito y no mayor que `MAX_TIMER_DELAY_MS`; de lo contrario, Node lo acota a un milisegundo. Mide solo la demanda pendiente, por lo que ningún temporizador corre mientras el código aguas abajo renderiza o espera antes de pedir el siguiente trozo. La primitiva sigue solo notificando, por lo que el transporte debe observar la señal estable; los adaptadores de DeepSeek y pi-ai demuestran que ese timeout cierra su cuerpo de respuesta real o la solicitud del SDK.

## Qué NO recibe timeout

Los `read`/`write`/`edit` de archivos locales no reciben `timeoutMs`: la E/S de archivos corre sin medir el tiempo porque un plazo mataría trabajo que el SO terminará de todos modos. Consulta [la página de subsistema de sistema de archivos](../../../docs/subsystems/filesystem.es.md).

## Experiencia del modelo

Indirectamente, a través de consumidores como `dsh-tool-call-timeout-policy`, que puede reemplazar un resultado de provider con un error de timeout retenido o suprimir un resultado tardío.

#### Efecto de KV Cache

Sin invalidación directa; el consumidor nombrado es dueño de cualquier cambio de prefijo de solicitud.

## Limitaciones conocidas y trabajo diferido

- **Solo notificación** — un plazo no puede detener el trabajo que ignora su señal; toda capacidad sigue necesitando su propia vía de terminación de socket/proceso/tarea.
- **`timeoutMs <= 0` es vocabulario interno** — desactiva el temporizador local solo después de que un backend propietario haya resuelto la política, nunca como una perilla pública del modelo o los plugins.
- **La primera razón de anulación gana la clasificación** — cuando una cancelación de upstream gana al temporizador local, esta capa no puede informar después de que su propio timeout también habría transcurrido.
- **Un idle watchdog no es un plazo total** — se rearma por cada demanda de iterador pendiente y excluye deliberadamente el tiempo de pensamiento del consumidor.
