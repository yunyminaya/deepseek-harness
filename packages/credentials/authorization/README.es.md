# dsh-authorization

[English](README.md) | Español

Service Definition de autorización (`ctx.authorization`). Algunas credenciales no pueden configurarse, solo obtenerse: conseguir una significa una conversación con un humano — abrir esta página, pegar ese código, elegir una cuenta. Este seam es dueño de esa conversación y del ciclo de vida que la rodea, y nunca del protocolo.

**Un flow es el conocimiento de un plugin sobre cómo obtener su propia credencial.** Se registra bajo el [`CredentialKey`](../credentials/README.es.md#two-key-spaces-two-questions) que escribe, así que un flow dice qué registro produce y, a través del scope de esa clave, qué plugin responde por el formato que contiene. Un segundo protocolo de autorización llega como otro flow, no como otro seam.

**El flow es dueño de la escritura.** Que `run()` se resuelva significa que el registro ya quedó confirmado a través de `ctx.credentials`; el seam confirma un commit que observó durante el intento — la mera presencia permitiría que una reautorización hiciera pasar un registro obsoleto por uno reciente — y rechaza un flow que se resolvió sin uno. Confirmar dentro del flow es lo que permite que una biblioteca que persiste a través de su propio adaptador de almacenamiento siga siendo el único escritor, en lugar de copiarse de vuelta y escribirse dos veces.

**La interacción viaja con la petición, no con un registro.** Quien inicia una autorización es quien puede hablar con el humano sobre ella, así que los prompts llegan exactamente a la superficie que los pidió y un llamador headless aporta una interacción que declina. No hay un provider ambiente que pueda faltar, ni duda sobre a cuál de dos páginas abiertas pertenece un prompt.

## Superficie

```ts
import type { Context } from '@deepseek-ai/cordis'
import { AuthorizationDeclinedError, type AuthorizationSession } from '@deepseek-ai/dsh-authorization'
import { credentialKey } from '@deepseek-ai/dsh-credentials'

declare const ctx: Context
declare const exchange: (signal: AbortSignal) => Promise<void>

const key = credentialKey('llm-pi-ai', 'openai-codex')

const dispose = ctx.authorization.registerFlow({
  key,
  label: 'ChatGPT (Codex)',
  methods: [{ id: 'oauth', label: 'Sign in with ChatGPT' }],
  async run(session: AuthorizationSession) {
    session.notify({ message: 'Continue in your browser', url: 'https://auth.example/start' })
    const code = await session.prompt({ kind: 'text', message: 'Paste the code' })
    // Commits the record through ctx.credentials before resolving.
    await exchange(session.signal)
    void code
  },
})

ctx.authorization.list()                    // [{ key, label, methods, inFlight }]
ctx.authorization.describe(key)             // the same entry, or undefined
await ctx.authorization.begin({             // { status: 'authorized' | 'cancelled' }
  key,
  interaction: { notify: () => {}, prompt: () => Promise.reject(new AuthorizationDeclinedError()) },
})
ctx.authorization.cancel(key)               // withdraw whatever is running for the key
dispose()
```

Un intento por clave a la vez. Un segundo llamador recibe `ALREADY_IN_FLIGHT` en lugar de sumarse, porque los dos estarían preguntando a humanos distintos a través de un mismo flow y el segundo contestaría las preguntas que le hicieron al primero. `inFlight` está en la entrada para que una superficie muestre el botón deshabilitado en lugar de descubrirlo por error.

`cancel(key)` existe junto a la señal de la propia petición porque un transporte de petición/respuesta responde a un botón Cancelar en una segunda llamada, sin tener ningún handle sobre la señal de la primera. Un flow cuya inscripción se elimina a mitad de intento se retira de la misma manera: su runner pertenece a un plugin que está desapareciendo.

Un intento cuyo llamador ya se retiró nunca reclama la clave ni inicia el flow — confiar en que cada flow compruebe su señal antes del primer await permitiría que uno que no lo hace se quedara colgado con la clave en la mano. La validación sigue ejecutándose primero, así que un llamador que nombra una clave o un método inexistente se entera de ello haya o no abandonado también.

El «no» de un humano es un resultado, no una avería. Una interacción que declina rechaza su prompt con `AuthorizationDeclinedError`, y un intento que falla después de un prompt declinado se resuelve como `cancelled`, exactamente igual que una señal retirada; cualquier otro rechazo de prompt sigue siendo un fallo del flow que llega al llamador. Un notice es de disparar y olvidar por el mismo principio, retenido en el seam: una superficie que no puede renderizarlo pierde el notice, nunca el intento.

`authorization/settled (key, settlement)` se dispara después de liberar la clave, para todo resultado terminal. `settlement` añade `failed` a los dos estados que `begin()` puede devolver: un fallo llega a su propio llamador como un error lanzado, así que el flujo de eventos es el único sitio donde un observador que no inició el intento puede distinguir un rechazo de una avería. Los fallos de los listeners están contenidos: cada listener se ejecuta, un throw o un rechazo se registra sin cambiar el resultado del intento terminado, y solo un fallo con código `INVARIANT` se vuelve a lanzar después de que el resto se haya ejecutado.

## El vocabulario de la interacción

Un notice es unidireccional y nunca lleva un secreto: un mensaje y, opcionalmente, la página que el humano debe abrir y el código que debe introducir allí. Un prompt es una pregunta que el flow no puede responder — `text`, `secret` o `select` — y `secret` solo se diferencia de `text` en la presentación. Un prompt lleva su propia `signal`, de modo que un flow que compite un código tecleado contra una callback del navegador puede retirar la pregunta perdedora mientras el intento continúa; la señal de la petición retira el intento entero en su lugar.

El vocabulario es deliberadamente más pequeño que el de cualquier provider individual: describe lo que una superficie debe renderizar, así que una superficie que renderiza un flow los renderiza todos.

## Experiencia del modelo

Ninguna, porque la autorización es una conversación de tiempo de configuración con un humano y ningún flow, notice ni prompt llega a una petición al modelo.

#### Efecto en la KV Cache

Sin invalidación; ningún estado de autorización entra en un prefijo de petición.

## Limitaciones conocidas y trabajo pendiente

- **Ningún flow es reanudable** — un intento vive en el proceso que lo inició, así que una recarga del navegador durante un inicio de sesión lo abandona y el humano empieza de nuevo. Los intentos durables necesitan un almacén que este seam no tiene.
- **Nada revoca** — cerrar sesión es `ctx.credentials.deleteRecord(key)`, que olvida el registro local sin avisar al emisor. Un provider que necesite una revocación en el servidor no tiene todavía dónde declararla.
- **Una clave sin flow es inerte** — el seam informa de lo que está registrado, así que un registro dejado por un plugin desinstalado puede eliminarse pero no reautorizarse. Reconocer ese huérfano es tarea del llamador, igual que con [`listRecords()`](../credentials/README.es.md#surface).
