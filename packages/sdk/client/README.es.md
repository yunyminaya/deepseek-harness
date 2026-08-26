# @deepseek-ai/dsh-sdk-client

[English](README.md) | Español

El SDK de cliente TypeScript para controlar un runtime de DeepSeek Harness como subproceso sobre JSON-RPC por stdio — el gemelo de diseño del [SDK de Python](../../../python/README.md) (`deepseek-harness`), que comparte el mismo peer de runtime, protocolo y capas: `DeepSeekHarness` es la API de alto nivel de run poseído, `HarnessClient` el cliente de protocolo de nivel inferior. La raíz del paquete enumera la interfaz de consumidor: las dos capas de cliente, los tipos orientados al caller y `JsonRpcResponseError`; los módulos de fuente, los helpers de normalización y la maquinaria de entrega de suscripciones no son imports de consumidor. Una librería pura: no registra nada en un contexto de Cordis; el proceso de runtime que lanza es un harness completo cuya composición decide su propio `cordis.yml`.

A diferencia del SDK de Python, la especificación de lanzamiento es totalmente explícita (`command`/`args`): este paquete es para consumidores TypeScript cercanos al repo — incluidos el backend [`dsh-subagent-dsh-sdk`](../../subagent/subagent-dsh-sdk/README.md) y la automatización — que saben qué runtime lanzan. La resolución de runtime empaquetado (encontrar un ejecutable incluido en el paquete) sigue siendo asunto de la distribución de Python.

## DeepSeekHarness

```ts
import { DeepSeekHarness } from '@deepseek-ai/dsh-sdk-client'

await using harness = new DeepSeekHarness({
  launch: { command: 'node', args: ['lib/bin.js', 'cordis.yml'] },
  provider: 'deepseek-official',
  model: 'deepseek-v4-flash',
  maxTokens: 49_152,
})
const result = await harness.run('say hi')
console.log(result.finalResponse)
```

El subproceso arranca de forma perezosa en el primer uso y sigue siendo propiedad de la instancia entre llamadas a `run()`; `close()` (o `await using`) es obligatorio para que el hijo siempre se recoja. `start()` memoiza el handshake de `initialize` (el cwd del workspace — resuelto absoluto antes de cruzar el wire — más la ruta provider/modelo y el tope de salida positivo opcional `maxTokens`); un handshake fallido recoge el runtime e intercambia un cliente nuevo, así que una llamada posterior reintenta con un subproceso nuevo (hasta `close()`, que es terminal). El tope se aplica a cada petición de root agent y lo heredan los descendientes en proceso; los plugins de compactación son dueños de sus límites de resumen separados. `session(id?)` abre un handle de sesión con nombre o nueva.

`run(input, { sessionId?, onNotification? })` es el dueño de un intervalo de actividad: encola el prompt, espera a que su `MessageId` aparezca en un recibo durable de `agent/inbox/spliced`, y luego recoge hasta el siguiente `idle` de todo el agent. Devuelve `RunResult { sessionId, finalResponse, events, notifications }`. `finalResponse` es el último texto de assistant de la root-session confirmado en ese intervalo, no una respuesta asignada causalmente al prompt; el steering, el contexto inyectado y otro trabajo encolado pueden contribuir antes del idle. `events` contiene los eventos de la root-session, mientras que `notifications` contiene además los descendientes descubiertos desde `subagent.started`, todo en orden de wire. El resultado no lleva ningún estado a nivel de prompt ni razón de turno. La pérdida de transporte, el timeout y las violaciones de protocolo rechazan; los resultados del modelo siguen siendo observables en el flujo de eventos sin atribuirse a una entrada.

## HarnessClient

El cliente de protocolo bajo la API de run poseído: `start()`/`initialize()`/`prompt()`/`request()`/`close()` explícitos, más suscripciones de notificación. `prompt()` devuelve el id del mensaje encolado en cuanto el runtime lo acepta; nunca espera actividad del agent. `subscribe(filter?)` devuelve una `NotificationSubscription` (`next()` awaitable, `tryNext()` no bloqueante, iteración async); `subscribeSessionTree(id)` limita el alcance a una sesión y a los descendientes descubiertos desde las aristas de linaje de `subagent.started` — el runtime notifica por cada sesión de su contexto, y el scoping es del lado del cliente, exactamente como en el SDK de Python. Las superficies de error están tipadas y se exportan desde este paquete: `JsonRpcResponseError` (respuesta de error de wire, con code/data conservados), `RequestTimeoutError` (se agotó un límite configurado), `SdkProtocolError` (una respuesta fuera del protocolo documentado), `TransportClosedError` (el runtime ya no está — el mensaje lleva el código de salida y una cola de stderr acotada).

`close()` solicita el `shutdown` del protocolo (acotado por `shutdownTimeoutMs`, por defecto 1000 ms) y luego recorre una escalera stdin-EOF → SIGTERM → SIGKILL (`disposeEofGraceMs` por defecto 6000, `disposeGraceMs` por defecto 3000) hasta que el proceso haya salido de verdad. La escalera es privada de este cliente: se ejecuta fuera de cualquier contexto de harness, así que no puede montarse en el servicio [`dsh-subprocess`](../../subprocess/README.md) — la excepción documentada del seam para transportes gestionados por SDK. Es idempotente, y un cliente cerrado rechaza su reutilización.

`HarnessClientOptions.env` reemplaza por completo el entorno del hijo cuando se da (`undefined` hereda el del padre); los callers son dueños de la política de credenciales — `scrubbedParentEnv` de `dsh-subprocess` es la base de saneamiento compartida para lanzamientos que buscan aislamiento.

## Experiencia del modelo

Ninguna, ya que es una librería de proceso de cliente; el modelo corre en el runtime lanzado, cuya experiencia es propiedad de los plugins que compone su `cordis.yml`.

#### Efecto en la caché KV

Ninguno; este paquete ni ensambla ni envía una petición de provider.

## Limitaciones conocidas y trabajo diferido

- **Sin resolución de runtime empaquetado** — los callers nombran el ejecutable del runtime explícitamente; el descubrimiento de ejecutables empaquetados sigue del lado de Python hasta que exista un consumidor de distribución TypeScript.
- **Sin cancelación a mitad de turno** — el wire no tiene método de cancelación de prompt; abandonar un turno significa cerrar el runtime (consulta las [Limitaciones conocidas](../protocol/README.md) del protocolo).
- **Sin resultado ni cancelación por prompt** — el `prompt()` de bajo nivel devuelve solo un recibo de encolado; el `run()` de alto nivel es el dueño de la recogida de recibo a idle, y abandonarlo significa cerrar el runtime.
- **Las notificaciones cliente→servidor y las peticiones servidor→cliente no están implementadas** en ninguno de los dos extremos del wire; el transporte las porta para futuros flujos de aprobación.
