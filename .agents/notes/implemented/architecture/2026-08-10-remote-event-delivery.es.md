# Agent Note: Entrega remota de eventos (ctx.remote.$on)

Status: implemented

English | [中文](2026-08-10-remote-event-delivery.zh.md)

## Problema

Las [llamadas a métodos dirigidas del Typert Gateway](../../implemented/architecture/2026-08-02-typert-remote-method-calls.es.md) cubren solo la forma petición/respuesta y dejan deliberadamente los flujos de eventos de Session y las interacciones con estado para diseños separados. Cada **push unidireccional Host→consumidor** sigue por tanto montado en el API Proxy legado.

El Host posee una familia de eventos unidireccionales cuyos payloads ya son JSON y cuya emisión nunca vincula un AgentScope: `agent-preset/selected`, `commands/change`, `credentials/reference-updated`, `llm/adapters-updated` y `settings/document-updated`. Alcanzar a un suscriptor de la UI tomaba cuatro saltos: el evento cordis del Host, una variante `HostFrame` escrita a mano más su rama zod en apiproxy, un puente escrito a mano en client/runtime que lo re-emitía como evento cordis del Client, y finalmente el `ctx.on(...)` del consumidor. Añadir un evento así editaba cinco lugares (unión de frames, unión zod, listener del host-stream, puente del client y una declaración `Events` duplicada del lado Client), y ni uno solo de ellos enunciaba un hecho nuevo: el nombre, el tipo del payload y el punto de emisión ya estaban todos declarados por la fusión cordis `Events` del paquete propietario.

Esa declaración duplicada además es **con pérdida**: el lado Client la reformula como `settings/changed(ns: string)`, aplanando un tipo con brand a un `string` desnudo — lo contrario del contrato de métodos remotos, donde un tipo de consumidor apunta al único símbolo canónico del paquete de negocio.

## Decisión

La superficie Remote del consumidor lleva un único verbo de suscripción unidireccional, `ctx.remote.$on(event, listener)`, dirigido por una allowlist y con reenvío literal:

- `packages/api/remotes/src/remote-events.ts` contiene la allowlist de eventos del Host reenviables, y es el único punto de control sobre lo que un consumidor puede suscribir. El `src/types.ts` a su lado deriva la proyección de tipos y llena el asiento de selección, manteniéndose type-only según la convención del paquete. Ambos archivos figuran en los `files` de **ambas** caras de este paquete, de modo que el bucle de reenvío del Host y la superficie de claves del consumidor leen una sola declaración.
- El nombre del evento en el wire **es** el nombre del evento cordis del Host (`settings/document-updated`) sin prefijo `host/`, y el payload **es** la lista de argumentos del Host, elemento por elemento, sin proyección, redacción ni renombrado.
- El portador reutiliza el host stream existente: `HostFrame` gana una variante envolvente, `host/remote-event`. Ningún nuevo downlink.
- Las **firmas** de eventos no obtienen una segunda tabla. Cada paquete propietario mueve su declaración cordis `Events` a su export `./types` client-safe y type-only, así ambas caras leen la misma declaración y el tipo del listener de `$on` es el propio `Events[Event]`. «Literal» se cumple entonces por construcción y no por demostración.
- De cordis se toma prestada solo la *forma del tipo*, no su sistema de eventos: la semántica de entrega, el registro de suscripciones y la contención de fallos pertenecen a Typert.

Cuando la firma de una entrada de `Events` alcanza un símbolo solo-Host (un Service, `Agent`, un Context), la respuesta es **dividir el código hasta que la entrada aterrice limpiamente en `./types`** — nunca una declaración a medio dejar en `index.ts`, y nunca un tipo sombra estructuralmente equivalente en `./types`. Ninguno de los cinco paquetes necesita eso aquí: sus entradas alcanzan solo `SettingsNamespace`, `SettingsUpdateSource`, `CredentialRef` y `SessionId`, todos tipos puros. El paquete agent-presets renombra su módulo de vocabulario previo a `preset.ts`, dejando el `types.ts` exportado dedicado a la declaración client-safe de eventos.

Los cinco eventos viajan por esta ruta, y sus variantes `HostFrame` dedicadas o alias del Client desaparecen. Los consumidores de modelo se suscriben directamente a ambas entradas del propietario, `llm/adapters-updated` y `settings/document-updated`; los consumidores derivados de preset se suscriben a `agent-preset/selected`. Los frames que sí proyectan o deduplican datos siguen dedicados: `host/workspace-changed`/`-removed`/`host/archived-sessions-changed` (derivación de vista más estado de dedup por conexión), y `host/session-added`/`-removed`/`host/session-status`/`host/agent-error` (proyección de live-objects o campos derivados al tiempo de frame).

`skills/change`, `tools/change` y `system-prompt/change` tienen la misma forma pero **hoy no tienen consumidor**; bajo «exigir propietario y necesidad actuales» quedan fuera de la allowlist y se registran aquí solo como el asiento de extensión.

### Contrato del consumidor (dsh-typert-protocol)

type-meta gana un **predicado de forma**, un **asiento de selección** y **un** miembro en `TypertClientRemote`. Sin código runtime:

```ts
import type { Events } from '@deepseek-ai/cordis'

/** Cordis events shaped for one-way remote delivery: no Scope binding, void return. */
export type TypertForwardableEvent = {
  [Event in keyof Events]: unknown extends ThisParameterType<Events[Event]>
    ? ReturnType<Events[Event]> extends void ? Event : never
    : never
}[keyof Events]

/** The Host assembly's forwarding selection; api/remotes' allowlist fills it, no other package does. */
export interface TypertRemoteEventSelection {}

/** `$on`'s legal keys: selected, and present in the current compilation face. */
export type TypertRemoteEvent = Extract<keyof Events, keyof TypertRemoteEventSelection>
```

```ts ignore-check
/** Subscribe to one forwarded Host event; the returned disposer belongs to the calling fiber. */
$on<Event extends TypertRemoteEvent>(event: Event, listener: Events[Event]): () => void
```

`Events` se resuelve por programa: el vocabulario completo del Host en el programa Host, lo que la cara Client pueda ver en el programa Client. El mismo predicado se cumple entonces en ambos lados sin arrastrar declaraciones del Host hacia el Client.

**La superficie separa el verbo del consumidor del relevo del portador**: los consumidores se suscriben con `$on`, y quien posea el sumidero de frames del Host entrega cada frame decodificado con `$dispatch`. No puede ser una función a nivel de módulo que cruce plugins del Client — la puerta de pureza del bundle client (`packages/client/tsdown.client.ts`) admite value imports solo desde los `PLATFORM_MODULES` implícitos más la línea base `PRELOADED_CLIENT_EXTERNALS`, las peticiones `dsh.client.external` del paquete, la capa wire `INLINE_SAFE`, y las contribuciones `/remote` generadas. Inlinearla copiaría `ClientRemoteService` al bundle runtime, dejando `instanceof` permanentemente false. Un método de servicio cordis es la forma de colaboración que esa puerta prescribe:

```ts ignore-check
$dispatch(event: string, args: readonly unknown[]): void
```

client/runtime — el propietario del sumidero de frames del Host — lo llama directamente, de modo que el frame llega a la tabla de suscripciones sin un evento intermedio que lo retransmita. El parámetro `event` es `string`, no `TypertRemoteEvent`: este es un límite de wire, y un nombre al que nadie se suscribió se descarta silenciosamente.

La entrega no comparte implementación con el sistema de eventos cordis: solo unidireccional, sin modos waterfall/bail/parallel/serial y sin concepto `@mode` (`ReturnType extends void` es la expresión estática de esa regla), sin vinculación `this`, sin `EventOptions`, `prepend` o prioridad. Los listeners corren en orden de registro, y uno que lance un error se contiene y se registra — jamás debe tirar la bomba de frames (la misma postura que `ConnectionController` ya aplica a sus sumideros).

### La allowlist: una declaración que ambas caras leen

`packages/api/remotes/src/remote-events.ts` figura en los `files` tanto de `tsconfig.host.json` como de `tsconfig.client.json`, y es el único hogar de la allowlist; `src/types.ts` deriva su cara de tipos:

```ts
// remote-events.ts — the value
export const API_REMOTE_FORWARDED_EVENTS = [
  'agent-preset/selected',
  'commands/change',
  'credentials/reference-updated',
  'llm/adapters-updated',
  'settings/document-updated',
] as const

// types.ts — the type face, derived
export type ApiRemoteForwardedEvent = typeof API_REMOTE_FORWARDED_EVENTS[number]

declare module '@deepseek-ai/dsh-typert-protocol' {
  interface TypertRemoteEventSelection extends Record<ApiRemoteForwardedEvent, true> {}
}
```

Reenviar un evento más es por tanto **una línea en ese array**: la proyección de tipos, la superficie de claves de `$on` y el bucle de reenvío del Host derivan de él. `ctx.remote.$on('slots/changed', …)` (evento local del Client) y `$on('skills/change', …)` (declarado pero no seleccionado) son ambos **errores de compilación**.

La cara Host añade una aserción de forma, vinculando el vocabulario de eventos del Host a ese mismo array:

```ts ignore-check
API_REMOTE_FORWARDED_EVENTS satisfies readonly TypertForwardableEvent[]
```

Es una sentencia de expresión y no una constante con nombre, que `noUnusedLocals` rechazaría (el prefijo de guion bajo exime solo a parámetros). Obliga tres cosas: que el **nombre sea real** (el predicado se keyed en `keyof Events`), que el evento **no vincule Scope** (`goal/changed` y parientes tienen un `ThisParameterType` distinto de `unknown` y caen — la expresión estática de «sin dependencia de AgentScope»), y que el evento sea **unidireccional** (un retorno no-`void`, es decir una forma waterfall/bail, cae).

**Que es «literal» no se demuestra en ningún sitio porque se cumple por construcción**: el tipo del listener de `$on` viene de la única declaración cordis `Events` en el `./types` del paquete propietario, y el reenvío del Host lee esa misma declaración. No hay una segunda declaración que pueda desviarse.

La seguridad JSON es una preocupación runtime: antes de reenviar, apiproxy valida cada argumento con el `isJsonValue` de `dsh-session` y **lanza un error ruidoso** cuando uno falla, porque eso es un error de composición de la allowlist y no entrada no confiable.

### Contrato wire (apiproxy)

```ts ignore-check
| { type: 'host/remote-event'; event: string; args: JsonValue[] }
```

La rama zod mantiene `args: z.array(z.unknown())`: el frame llega de `JSON.parse`, así que cada elemento ya es un valor JSON, y el contrato estructural pertenece a la declaración `Events` del paquete propietario — la misma postura que el frame `session/projection` existente toma con su `value`.

`events.host()` se suscribe por allowlist cuando el stream abre. Cada stream es dueño de sus disposers, así que no hace falta ni un conjunto de difusión ni un listener de invalidación derivado.

`api/events.ts` es un archivo de contrato wire que el lado browser también compila, así que cada tipo que referencia debe venir de un **subpath client-safe y type-only** de un paquete propietario, jamás de la raíz del paquete. Evidencia: importar un tipo desde la raíz de `@deepseek-ai/dsh-session` arrastra al `declare module 'cordis' { interface Context { sessions: SessionStore } }` de la raíz hacia la cara de compilación Client y sobrescribe el `ctx.sessions: ISessions` del Client, produciendo 18 errores en los no relacionados `ui-input-trigger` y `ui-conversation`. `JsonValue` necesita por tanto un re-export desde `dsh-session/src/types.ts`.

### Los e2e de browser de apps/web pertenecen a la cara Host

Los e2e de `apps/web/tests/**` se type-checkean en el **`tsconfig.host.json`** de la raíz: arrancan un harness real in-process y leen `ctx.apiProxy`, el `get`/`create`/`flush` del `SessionStore` del Host, y `ctx.sessionProjectionCache`. **Conducir un browser en runtime no convierte un archivo en parte del programa Client** — moverlos al agregado Client produce inmediatamente 21 errores, porque un programa no puede sostener las fusiones de ambas caras para la misma clave de Context.

De ahí una disciplina de la que este diseño depende: **cuando esos tests importan un valor o un tipo de un paquete Client, arrastran el proyecto completo de ese paquete — y cada proyecto que este referencia — al grafo de build del Host**. Cuatro consumidores (`ui-settings-general`, `ui-settings-models`, `ui-permission`, `ui-commands`) referencian la cara Client de `api/remotes`, y esa cara no puede compilar hasta que el tsdown del Host haya generado `@deepseek-ai/dsh-goal/remote`. El resultado es un interbloqueo de orden de build: el tsc del Host necesita la cara Client, que necesita el artefacto generado, que el tsdown del Host produce después del tsc del Host.

Los pocos símbolos propiedad del Client quedan por tanto **espejados** del lado del test (`scaffold.ts` exporta las constantes espejo del welcome-notice; los dos e2e de chat siguen importando `dsh-client-runtime/client` porque el proyecto `runtime` ya está en el grafo del Host), lo que permite a esos cuatro consumidores salir del grafo del Host. Las 15 referencias a proyectos Client en `apps/cli/tsconfig.json` perdieron su rol de owner-map y desaparecen. Cada valor espejo coincide verbatim con su fuente; una desviación aparece como un selector fallido o un aviso no suprimido, ambos fallos ruidosos.

### Inventario de cambios

| Lugar | Cambio |
|---|---|
| `dsh-typert-protocol` | `src/types.ts` gana `TypertForwardableEvent`, `TypertRemoteEventSelection` y `TypertRemoteEvent`; `TypertClientRemote` gana `$on` y `$dispatch`. Solo tipos, sin runtime |
| mitad Client de `api/gateway` | `ClientRemoteService` implementa `$on` (suscripciones dirigidas por registro, propiedad `ctx.effect` para la fiber que llama) y `$dispatch` (entrega sobre instantánea en orden de registro, conteniendo un listener que lance o rechace) |
| `api/remotes` | Nuevo `src/remote-events.ts` (el valor de la allowlist) y `src/types.ts` (proyección de tipos, asiento de selección), ambos listados en los `files` de ambas caras; un export `./types` con `lib/types/**/*.js` añadido a `files`; la cara Host añade la aserción de forma y `import type {}` para los cinco `./types` propietarios; la mitad Client re-exporta esos cinco más `@deepseek-ai/dsh-api-gateway/client` |
| `tsconfig.base.json` raíz | Entradas `paths` client-safe para settings, credentials, llm, agent-presets y api-remotes types apuntan al plano **fuente** |
| `dsh-commands` / `dsh-settings` / `dsh-credentials` / `dsh-llm` / `dsh-agent-presets` | Cada miembro `interface Events` reenviado vive en el `./types` client-safe del propietario; agent-presets mueve su vocabulario de dominio previo a `preset.ts` para que el archivo exportado siga siendo `types.ts` |
| `host/apiproxy` | `HostFrame` gana `host/remote-event` y pierde las cinco variantes passthrough o de invalidación dedicadas con sus ramas zod; `events.host()` se suscribe por allowlist y valida con `assertJsonArgs` |
| `dsh-session` | `src/types.ts` re-exporta `JsonValue` para que los archivos de contrato wire usen el subpath client-safe |
| `client/runtime` | Las cinco ramas de puente de eventos del Client colapsan en `ctx.remote.$dispatch(frame.event, frame.args)`, añadiendo una inyección `remote` y borrando sus declaraciones `Events` duplicadas |
| Siete consumidores | ui-commands / ui-model-selection / ui-settings-models / ui-settings-general / ui-permission / ui-agent-preset / ui-skill se suscriben vía `ctx.remote.$on(...)`, siguiendo el precedente de `ui-goal` para la importación de fachada type-only y la inyección `'remote'` |
| `client/connection` | El `emitHost` del fixture produce `host/remote-event` |
| `apps/web/tests` + `apps/cli` | Símbolos Client espejados del lado del test (ver arriba); `apps/cli/tsconfig.json` suelta sus 15 referencias a proyectos Client |

## Alternativas consideradas

**Abrir un canal downlink general para eventos Remote** (la contraparte push de `ctx.connection.rpc`, un tercer WebSocket). Es lo que mejor encaja con «Connection posee el portador, el Gateway jamás toca transporte», pero significa un stream nuevo en el downlink del Host, `WebApiClient`, `ConnectionController`, el fixture y los e2e web — un costo desproporcionado para este cambio. Reutilizar el host stream cuesta una tenencia temporal dentro de una unión de frames legada; cuando ese stream se mude, la envolvente se muda con él y el contrato del consumidor no cambia.

**Declarar un `TypertRemoteEventMap` separado en type-meta y dejar que los paquetes propietarios fusionen en él.** El conjunto de claves del consumidor igualaría exactamente «eventos declarados entregables remotamente», pero cada firma se escribiría una segunda vez fuera de cordis `Events`, exigiendo una prueba `extends` bidireccional para evitar que ambos se desvíen, más una dependencia type-meta nueva para tres paquetes propietarios. Compartir la única declaración `Events` hace esa equivalencia estructural, así que la tabla no se crea.

**Que el generador typert proyecte las declaraciones `Events` del Host** (codec, `.d.ts`, declaration map, como `/remote`). El generador ya analiza eventos del Host, pero no puede ver la intención de proyección o redacción, y cambiaría el generador y la superficie de build. El reenvío literal no necesita proyección.

**Dar a los eventos reenviables una función de proyección de payload** (una tabla de reenvío `{ name, project, zod }`). Podría plegar las dos entradas del directorio de modelos en una invalidación derivada y también cubrir la derivación de vistas de workspace, al costo de alinear a mano la lógica de proyección con los tipos de payload — la tabla central que el lado de métodos acaba de eliminar.

**Mover los e2e de browser de apps/web al agregado Client.** «Los tests del Client pertenecen a la cara Client» parece correcto y falla inmediatamente con 21 errores: esos tests usan servicios del Host, y en el programa Client `ctx.sessions` es `ISessions`.

**Dividir `directory-picker-browse`/`-native` en caras Host y Client** para que ningún paquete Client alcance el grafo del Host. La dirección es correcta — son genuinamente paquetes de doble mitad sin dividir — pero el cambio aterriza en paquetes de otro propietario y compra solo un grafo de build más limpio; una vez que este diseño espeja los símbolos Client del lado del test, ya no necesita la división. **Evaluado y descartado.**

## Verificación

Lo que fija este comportamiento:

- Un test de composición real pone un frame `host/remote-event` en el host stream real por cada emisión del Host, con `event` el nombre del Host y `args` iguales elemento por elemento.
- Los negativos a nivel de tipo rechazan tres clases candidatas: un nombre que no es evento, un evento vinculado a Scope (`goal/changed`) y un evento cuyo retorno no es `void`. `$on('slots/changed', …)` (local del Client) y `$on('skills/change', …)` (declarado pero no seleccionado) no compilan, así que la superficie de claves de `$on` equivale a la allowlist.
- Del lado del consumidor, `$on('settings/document-updated', …)` resuelve `ns` como `SettingsNamespace`: el brand sobrevive al wire.
- El disposer de `$on` pertenece a la fiber que llama, y dos registros de un mismo objeto función se retiran de forma independiente — una tabla keyed por identidad del listener los colapsaría, así que las suscripciones se dirigen por registro.
- La entrega contiene un listener que lanza Y uno que rechaza una promesa retornada: el retorno declarado es `void`, así que nadie espera un listener async, y su rechazo de otra forma escaparía por completo a esta contención. La entrega itera una instantánea, de modo que suscribirse o disponer a mitad de frame no cambia quién recibe ese frame.
- `assertJsonArgs` se prueba unitariamente de forma directa y no emitiendo un emit malformado a través del bus: un `ctx.emit` tipado no puede construir uno, ya que cada evento de la allowlist tiene un payload estáticamente JSON-safe.
- Las cinco variantes `HostFrame` dedicadas, los cinco alias del lado Client y sus ramas de puente están ausentes. Los directorios de modelos observan ambas entradas del propietario, mientras que los consumidores de fila de comando, skill y sesión observan el evento de selección comprometida del propietario del preset.

## Consecuencias

- **Tenencia dentro de una unión de frames legada.** El contrato vive en el `HostFrame` de apiproxy, así que un lector puede asumir que apiproxy posee los eventos Remote. El JSDoc del frame nombra a `api-remotes` como propietario de la allowlist, y el README de apiproxy registra la tenencia bajo limitaciones conocidas. Cuando el host stream salga de ese paquete, la envolvente se muda con él y el contrato del consumidor no cambia.
- **Dos archivos rompen el contrato de disjointness de caras de api/remotes.** `src/remote-events.ts` y `src/types.ts` pertenecen a ambos proyectos, así que cada uno emite una declaración idéntica al `lib/types` compartido. El contenido es byte-idéntico y los `.tsbuildinfo` permanecen separados, así que esto es inofensivo en la práctica; la sección de límites de build del README enuncia la excepción y su causa (la entrada `paths` apunta a la fuente).
- **El relevo del portador es visible para el desarrollador.** Cualquier plugin Client que sostenga `ctx.remote` puede llamar `$dispatch` y sintetizar un evento reenviado. Esa exposición es anterior al verbo — `ctx.emit` era igualmente alcanzable mientras un evento interno retransmitía el frame — y equivale a lo que `connection/reset` ya permite para una reconexión fabricada; el Client es un único dominio de confianza. Los tests fijan la conversión relevo→`$on` y no pretenden que el puerto autentique a su llamador.
- **Un argumento malformado falla en la contención del emisor, no en la carga.** `assertJsonArgs` lanza dentro del listener de reenvío, así que la contención de listeners del seam emisor lo registra y descarta ese frame: ruidoso en el log del Host y no en la carga ni en el punto de emisión.
- **Los valores espejo del test pueden desviarse.** Nada comprueba mecánicamente las constantes Client espejadas en `apps/web/tests` contra su fuente; la red de seguridad es solo que una desviación pierde un selector. La regla vive en `apps/web/tests/README.md` y la sostiene la revisión — una puerta a nivel grep se consideró y se descartó deliberadamente.
- **Capacidades cedidas.** Sin payloads proyectados ni redactados, sin eventos vinculados a Scope (`agentCtx.remote.$on`) y sin replay al reconectar — estas son señales puras de invalidación, y `connection/reset` ya cubre el refetch tras una reconexión. Los eventos de sesión del stream mux, los frames exigibles y las líneas base de instantánea quedan fuera de alcance.
- **Los paquetes Client permanecen en el grafo del Host.** Doce proyectos (`connection`, `runtime`, `ui-slots` y parientes) todavía lo alcanzan a través del par sin dividir `directory-picker-browse`/`-native` y `api/gateway → client/connection`. Compilan y ya no implican la cara Client de api/remotes, así que no bloquearon este cambio; dividir esos paquetes eliminaría algunos pero se evaluó y descartó. Los dos e2e de chat que importan `dsh-client-runtime/client` dependen de que `runtime` ya esté en ese grafo — incidental, no una garantía.
- **El compañero de invariantes no sostiene chequeo runtime.** Una revisión anterior asertaba la forma del dispatch (`thisArg === null`, `mode === 'emit'`) sobre el bus de eventos vivo, lo que acoplaba el compañero al valor de la allowlist y hacía que rolldown lo elevara a un tercer chunk del bundle que la lista mecánica de publicación no lleva. La aserción `TypertForwardableEvent` de la cara Host ya rechaza ambas desviaciones en tiempo de compilación, así que el compañero es un installer vacío explicado.
