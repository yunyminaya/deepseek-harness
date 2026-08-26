# `@deepseek-ai/dsh-llm-retry`

[English](README.md) | Español

Plugin de funciones que aplica una política de reintentos específica del provider a través del waterfall (cascada de eventos) de pasos cerrados `agent/request-error` del agent loop (bucle del agente). No envuelve `ctx.llm.stream()`: cada llamada al adaptador sigue siendo un intento del provider, y cada reintento abre un turno numerado nuevo.

Cada adaptador de provider posee un `retryPolicy` anidado opcional, capturado cuando su ruta se registra en `ctx.llm` y transportado con cada llamada que alcanza la frontera del adaptador final de ese registro. Un fallo en vuelo conserva esa política en servicio si la ruta se elimina o se sustituye después; un fallo anterior a la selección de cualquier adaptador final no tiene política de provider y delega. La omisión usa el modo normal: cinco reintentos para `EMPTY_RESPONSE`, `RATE_LIMIT`, `SERVER`, `TIMEOUT` y `TRANSPORT`, con backoff exponencial acotado de 500 ms a 10 segundos y un 10 % de jitter. `EMPTY_RESPONSE` es la clasificación que los adaptadores hacen de una finalización degenerada del provider que no produjo contenido duradero, por lo que repetirla es seguro. Una política normal puede cambiar su presupuesto finito, los códigos elegibles y el backoff. El modo always pide primero la recuperación al downstream y después reintenta cada fallo de petición de modelo sin límite de intentos; el éxito, la cancelación o la eliminación del plugin lo detienen una vez que la recuperación delegada activa alcanza la quiescencia.

Ambos modos usan backoff exponencial acotado con jitter simétrico. Un `providerRetryAfterMs` válido igual o inferior a `maxDelayMs` sustituye al backoff local sin jitter. Un retraso del provider por encima del límite hace que el modo normal delegue, mientras que el modo always usa su backoff local configurado para no poder terminar por esa instrucción.

Antes de esperar, el plugin añade un evento `llm/retry` no superficial con el `retryId` compartido, el provider, el modo, la clave canónica de la política resuelta, el fallo y el retraso programado. Su payload está disponible desde la subruta `@deepseek-ai/dsh-llm-retry/types`, segura para el navegador, de modo que los renderizadores remotos pueden consumir el estado duradero sin cargar el runtime de la política. La clave incluye todos los campos que afectan al comportamiento y ordena los códigos del modo normal porque la elegibilidad usa pertenencia a conjuntos. Los números de reintento solo continúan entre eventos con el mismo provider y la misma clave completa de política, de modo que una sustitución de ruta con otros límites, otra pertenencia de códigos u otro backoff inicia su propia historia. Los eventos normales incluyen el máximo finito; los eventos de always lo omiten y las UIs muestran `∞`. Cuando la espera termina, el plugin añade `llm/retry-started` con el mismo `retryId`, turno, paso y número de reintento inmediatamente antes de devolver `{ kind: 'retry' }`; la cancelación durante el backoff no escribe ningún evento de inicio. El loop cierra entonces el turno fallido y abre un turno de reintento sobre la misma historia duradera. La cancelación y la eliminación del plugin abortan el backoff activo, drenan la recuperación delegada activa antes de aplicar el abort y hacen que un callback capturado antes de la eliminación falle en modo cerrado (fail closed).

El compañero `./invariant`, publicado por separado, comprueba que cada reintento programado nombra el turno abierto actual y el último paso cerrado, coincide con el provider duradero de la petición fallida, lleva identidades no vacías de provider y política, tiene límites específicos del modo, un registro de paso único, el número de reintento correcto de la política del provider y un retraso de timer acotado. También exige que cada evento `llm/retry-started` nombre un intento programado anterior con el mismo `retryId`, turno, paso y número de reintento, y rechaza los eventos de inicio repetidos. El jitter completo puede programar cero milisegundos en su límite inferior.

```yaml
- name: '@deepseek-ai/dsh-llm-deepseek'
  config:
    apiKeyEnv: DEEPSEEK_API_KEY
    retryPolicy:
      mode: always
      backoff:
        initialDelayMs: 1000
        maxDelayMs: 30000
        jitterRatio: 0.2

- name: '@deepseek-ai/dsh-llm-retry'
```

El ejecutor no tiene configuración de política. Los adaptadores multiprovider como `dsh-llm-pi-ai` colocan `retryPolicy` dentro de cada perfil de provider, evitando una segunda lista de nombres de provider.

## Experiencia de modelo

### Recuperación de peticiones de modelo

#### Lo que ve el modelo

Ningún evento de reintento, retraso, error de provider ni salida parcial fallida es visible para el modelo. El turno de reintento reconstruye la misma petición explícita de provider/modelo a partir de la historia duradera de la superficie, salvo que una política de recuperación del downstream cambie deliberadamente esa superficie; los chunks fallidos nunca entran en los mensajes derivados.

#### Efecto en tokens

Cada reintento es una nueva petición al provider y puede repetir la facturación de tokens de entrada. El modo normal tiene un presupuesto finito; el modo always puede consumir peticiones sin límite hasta el éxito o la cancelación. `llm/retry` en sí no aporta tokens.

#### Efecto en la caché KV

La petición reconstruida conserva el prefijo anterior y es elegible para la reutilización de la caché del provider según las reglas de ese provider. El evento de reintento no superficial no cambia la identidad de la caché.

## Limitaciones conocidas y trabajo diferido

- **Los turnos de agent son la única frontera de reintento** — los consumidores directos de `ctx.llm.stream()` siguen siendo de un solo intento porque un stream en bruto no puede separar de forma duradera los chunks ya emitidos.
- **El modo always reintenta fallos permanentes** — los errores de autenticación, cuota, petición no válida, protocolo y contexto irrecuperable continúan hasta el éxito, la cancelación o la eliminación; los despliegues controlan el coste y la latencia específicos de cada provider.
- **Los presupuestos finitos del plugin se suman** — el modo normal solo cuenta sus códigos configurados y la política exacta del provider, mientras que la compactación por desbordamiento de contexto tiene su propio presupuesto. Cualquier política solapada debe definir el comportamiento del orden de registro.
- **Las políticas de recuperación se componen por orden de waterfall** — el modo always acepta un reintento del downstream antes de aplicar su fallback. Una política posterior que ignora la cancelación y nunca se asienta impide también que el fallback, la quiescencia de turno y la eliminación del plugin lleguen a completarse.
- **`llm/retry` registra la programación, no la finalización** — los eventos posteriores de paso y turno establecen el éxito, el agotamiento o la cancelación.
