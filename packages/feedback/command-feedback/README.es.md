# @deepseek-ai/dsh-command-feedback

[English](README.md) | Español

Feedback de sesión independiente del desencadenante más captura `/feedback` orientada a humanos. El paquete exporta `recordFeedback(session, text)`, que añade un evento `feedback/record` solo de registro. Su plugin registra un comando global a través de [`ctx.commands`](../../interaction/commands/README.es.md), de modo que todos los adaptadores de comando compuestos lo descubren; el cliente web incluido lo ejecuta sin un turno de modelo.

## Contrato de comando

| Entrada | Resultado |
|---|---|
| `/feedback <text>` | Añade `feedback/record` y acusa recibo con `Feedback recorded for session {sessionId}`, `Anonymous user: {userId}`, más la divulgación de compartición de sesión. |
| `/feedback` | Devuelve un error de uso directo. La entrada de solo espacios en blanco se trata como vacía. |

El espacio en blanco circundante se descarta, pero el feedback no se analiza de ninguna otra forma: sin truncamiento, sin cambio de mayúsculas ni palabras de control. El texto que parece otro comando, como `/feedback /plan felt slow`, es contenido de feedback. Los comandos repetidos producen cada uno su propio evento; nada se reemplaza ni se fusiona.

## Divulgación de compartición de sesión

El acuse nombra el id de la sesión receptora e informa de cómo se comparte esa sesión, leído del servicio [`telemetry`](../../session/session-telemetry/README.es.md) montado a través del contexto del plugin (`ctx.get('telemetry')`, nunca una inyección declarada). La divulgación es una frase elegida del [`SessionTelemetrySharingStatus`](../../session/session-telemetry/README.es.md) del backend:

| Estado divulgado | Frase del acuse |
|---|---|
| `full` | `Session sharing is enabled.` |
| `feedback-only` | `Session sharing is feedback-gated; recording feedback releases the session prefix for sharing.` |
| `disabled` | `Session sharing is disabled.` |
| sin servicio | `Session sharing is not configured.` |

La divulgación declara únicamente la política de compartición actual del despliegue; nunca promete entrega ni retención. Con `full` o `feedback-only`, los registros se entregan a la cola no bloqueante del backend y el SDK es dueño del agrupamiento, los reintentos y la política de pérdida, así que la frase no afirma nada sobre lo que llegó a un recopilador; `disabled` no afirma nada sobre una reconfiguración futura. La divulgación no añade ningún evento y nunca entra en la superficie del modelo.

## Lo que este plugin hace y no hace

`recordFeedback(session, text)` es la vía de escritura independiente del comando. Rechaza el texto normalizado vacío y añade `feedback/record { text }`; otra interfaz, hook o integración de host puede llamarlo sin construir un comando de barra. El manejador de `/feedback` usa ese productor y no inicia ningún trabajo de modelo. El Consumer opcional [`dsh-session-telemetry-otel`](../../session/session-telemetry-otel) observa el evento sin cambiar su contrato de captura.

El texto de feedback aparece en exactamente una carga útil durable: `feedback/record`. [`dsh-commands`](../../interaction/commands/README.es.md) sigue añadiendo su par genérico `command/run` / `command/done`, pero esta definición fija `recordInput: false`, así que `command/run` omite `args`; el `command/done` emparejado lleva solo el resultado. Los tres eventos son solo de registro y están ausentes de la superficie ordenada, de `deriveMessages()` y de las peticiones del modelo. Estas adiciones inician el drenaje ansioso ordinario de la persistencia, pero ningún productor fuerza `session/flush`, de modo que el acuse significa que el feedback está en el registro, no que ha llegado al disco. El acuse identifica tanto la sesión receptora como el [usuario anónimo compartido](../../identity/anonymous-user-id/); el primer feedback aceptado para un home de harness puede crear `$DSH_HOME/.anonymous-user-id`. La entrada vacía rechazada solo deja el par de comando resuelto como `kind: 'error'`, sin `feedback/record` y sin búsqueda de id de usuario.

El evento es la autoridad en lugar del registro del comando porque el feedback puede llegar a través de un desencadenante distinto de `/feedback`. Mantener la carga útil fuera de `command/run` evita que dos registros lleven el mismo texto.

## Composición

El productor inyecta solo `commands`. Una app personalizada monta el registro más este plugin:

```yaml
- id: commands
  name: '@deepseek-ai/dsh-commands'
- id: command-feedback
  name: '@deepseek-ai/dsh-command-feedback'
```

La base `dsh` incluida monta este comando incondicionalmente; no tiene configuración ni dependencia de la pila de objetivos persistidos. El cliente web lo expone a través del adaptador de comando. El modo headless, la automatización ACP y JSON-RPC no proporcionan un adaptador de comando, así que no lo exponen.

## Model Experience

### Captura humana de `/feedback`

#### Lo que ve el modelo

Nada. La entrada de barra, `feedback/record` y el acuse están ausentes de las peticiones del modelo. El evento de feedback y los registros del ciclo de vida del registro son solo de registro y no llevan `surfaceOp`, así que nunca llegan a la superficie ordenada, a `deriveMessages()` ni a un system prompt. Registrar feedback durante un turno no cambia las peticiones restantes de ese turno.

#### Efecto de tokens

Cero efecto directo de tokens. Ni una entrada aceptada ni un error de uso añaden tokens de modelo, en el turno de registro ni en ningún otro posterior.

#### Efecto de KV Cache

Independiente de la ruta de petición del modelo. El registro solo añade al registro de sesión, dejando intacto un prefijo de petición ya reutilizable. Nada de lo que contribuye este paquete puede invalidar la reutilización de la caché.

## Limitaciones conocidas y trabajo diferido

- **No hay superficie de recuperación ni de gestión de feedback** — el plugin OTel opcional usa el evento solo como desencadenante de compartición. No hay recuperación, agregación, categorización ni herramienta orientada al modelo para `feedback/record`.
- **Sin campos estructurados** — una entrada es una sola cadena de texto libre sin categoría, severidad ni enlace a evento referenciado, así que el feedback no se puede filtrar por asunto sin releer su texto.
- **Sin corrección ni retirada** — el registro de sesión es solo de adición y este paquete no añade ningún tombstone, así que una entrada errónea queda registrada y solo puede ser superada por una posterior.
- **Sin barrera de durabilidad explícita** — el acuse sigue a la adición, no a un flush, así que una entrada registrada justo antes de un cuelgue puede perderse con cualquier otra cola sin vaciar. El feedback no merece forzar una escritura síncrona en disco; un Consumer que la necesite espera `ctx.sessions.flush(session)`.
- **Sin acuse visible en una sesión nueva** — el transcript web renderiza las filas de comando solo una vez que la sesión está activa, así que `/feedback` en una sesión aún en blanco registra el evento pero no muestra ninguna fila de acuse. Registrar feedback después del primer mensaje se renderiza con normalidad.
- **Solo web entre los puntos de entrada incluidos** — el modo headless, la automatización ACP y JSON-RPC no proporcionan un adaptador de comando, así que `/feedback` no está disponible allí.
