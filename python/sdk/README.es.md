# DeepSeek Harness Python SDK

[English](https://github.com/deepseek-ai/deepseek-harness/blob/master/python/sdk/README.md) | Español

SDK de Python por subproceso para manejar DeepSeek Harness a través de JSON-RPC por stdio. El
runtime hereda las variables de entorno habituales de DeepSeek Harness, como
`DEEPSEEK_BASE_URL` y `DEEPSEEK_API_KEY`, de modo que los llamantes pueden usar directamente
endpoints de modelo reales o apuntar esas variables a un proxy local.

Instala la distribución `deepseek-harness-sdk` desde PyPI; el módulo de importación sigue siendo `deepseek_harness`:

```sh
python -m pip install deepseek-harness-sdk
```

Instalar `deepseek-harness-sdk` instala el wheel de plataforma `deepseek-harness-runtime-bin` de la misma versión exacta. Por tanto, el punto de entrada habitual no necesita ningún argumento de ejecutable:

```py
from deepseek_harness import DeepSeekHarness

with DeepSeekHarness() as harness:
    result = harness.run("Say hi.")
```

`DeepSeekHarness` conserva su subproceso de runtime arrancado de forma diferida (lazy) para reutilizarlo entre llamadas. Úsalo como gestor de contexto, como arriba, o llama a `close()` explícitamente al terminar.

Por defecto, el SDK lanza el ejecutable de un solo archivo `dsh-jsonrpc-agent` incluido en el paquete `deepseek-harness-runtime-bin` e inyecta la configuración predeterminada de ese paquete (el servidor JSON-RPC por stdio, el agent core, el adaptador DeepSeek precargado, la persistencia de sesiones JSONL con una política de checkpoints semánticos compuesta explícitamente, bash local) mediante `DSH_CORDIS_CONFIG`. Para ejecutar una composición de plugins propia, conserva la entrada `@deepseek-ai/dsh-sdk-jsonrpc-server` en la configuración y pasa la ruta de la configuración Cordis.

```py
from deepseek_harness import DeepSeekHarness

with DeepSeekHarness(
    provider="deepseek-official",
    model="deepseek-v4-flash",
    max_tokens=49_152,
    cordis="examples/jsonrpc-agent/cordis.yml",
) as harness:
    result = harness.run("Make the requested code change.")
```

`provider` selecciona una ruta de provider registrada por la composición Cordis elegida; `model` es el id de modelo que resuelve ese adaptador. `max_tokens` es un tope opcional y positivo de tokens de salida por petición para el agent raíz y sus descendientes en proceso; si se omite, el provider por defecto queda al mando. Los resúmenes de compactación conservan el límite independiente que configura su plugin de compactación. La composición predeterminada incluida registra `deepseek-official`. Una composición propia puede montar `llm-pi-ai`, configurar allí credenciales/endpoints específicos del provider y seleccionar cualquier provider/modelo presente en el catálogo instalado de pi-ai.

El [tutorial del SDK de Python](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/guide/python-sdk.md) ofrece una ruta ordenada de instalación y primera ejecución sin la Web UI. El [ejemplo `jsonrpc-agent`](https://github.com/deepseek-ai/deepseek-harness/blob/master/examples/jsonrpc-agent/README.md) es dueño del archivo Cordis independiente completo que se usa allí.

`Session.run()` es dueño de un intervalo de actividad que va desde la recepción duradera de su prompt en la bandeja de entrada hasta el siguiente idle (inactividad) del agent completo, y devuelve `RunResult(session_id, final_response, finish_reason, events, notifications, session_root)`. `final_response` es el último texto de asistente confirmado de la sesión raíz dentro del intervalo. `finish_reason` es el `kind` del último `turn/end` de la sesión raíz en el intervalo, como `completed`, `max-tokens` o `error`, y es `None` cuando ningún turno terminó. Un `turn/end` sin un `data.reason.kind` de tipo string viola el protocolo del runtime y lanza `SdkProtocolError`. Ambos campos del resultado describen el intervalo propiedad de la llamada, no una salida o un final atribuido causalmente al prompt. El steering (direccionamiento), el contexto inyectado y otro trabajo en cola pueden contribuir antes del idle.

`HarnessClient` conserva el linaje de subagentes descubierto durante toda la vida del proceso de runtime. Durante cada `Session.run()`, `RunResult.notifications` y `on_notification` reciben la sesión raíz y todas las notificaciones conocidas de los descendientes en orden de wire, incluidos los eventos de ciclo de vida y de sesión de subagentes anidados. `RunResult.events` contiene solo eventos de la sesión raíz, de modo que los mensajes de los descendientes no pueden sustituir la respuesta raíz. La `session_prompt()` de bajo nivel devuelve el `MessageId` encolado inmediatamente; los llamantes que omiten `Session.run()` son dueños ellos mismos de cualquier límite de actividad posterior.

El mismo comportamiento puede seleccionarse para el subproceso de runtime con `DSH_CORDIS_CONFIG`. La inyección vive en `HarnessClient.start()`, de modo que el lanzamiento por defecto del cliente de bajo nivel también la recibe: cuando el lanzamiento resuelve al runtime empaquetado y no se ha establecido ni `cordis` ni un `DSH_CORDIS_CONFIG` no vacío (el runtime trata un valor vacío como ausente, y la comprobación de inyección también), se usa la configuración predeterminada incluida; un `runtime_bin`, `bridge_bin` o `launch_args_override` explícitos desactivan la inyección por completo. Ver el [README del sdk-runtime](https://github.com/deepseek-ai/deepseek-harness/blob/master/python/sdk-runtime/README.md) para los portadores del runtime (exe de producción frente a closure node de solo desarrollo) y cómo obtenerlos.

`cwd` y `runtime_cwd` se resuelven a rutas absolutas antes del lanzamiento del subproceso, de la inyección de entorno y del handshake del wire. La API pública expone solo las opciones aplicadas: la persona de despliegue y la persistencia pertenecen a `cordis.yml`, mientras que `session_root` sigue siendo la comodidad de alto nivel que establece `DSH_SESSION_ROOT`.
