# API Gateway

[English](api-gateway.md) | Español

Esta es la referencia de estado actual del Typert API Gateway. Describe cómo los servicios de negocio declaran métodos Remote unarios, cómo la compilación genera los contratos de Host y de Client, y cómo las llamadas reutilizan la RPC de Connection y la ruta `/api`. Los eventos de sesión, los datos incrementales y otros protocolos de streaming quedan fuera del alcance de este documento; pueden usar la misma Connection, pero no utilizan descriptores de método Remote.

## Modelo de programación

Los servicios de negocio usan `@Remote` o `@RemoteScope` para seleccionar los métodos expuestos al Client. Los métodos sin marcar no entran en los tipos de Client generados ni en las contribuciones de runtime, y no se pueden llamar a través de `ctx.remote`.

`@Remote` denota la llamada a un servicio de Cordis registrado en el Context raíz del Host. Los objetos Host complejos no pueden cruzar el cable directamente; el paquete de negocio debe declarar su asociación con una identidad de cable mediante `TypertLookupMap` y registrar un provider de resolución predeterminado con `ctx.typert.lookups` en runtime. Por ejemplo, un parámetro `Agent` llamado `agent` en la firma del Host produce un campo de cable `agentId`, y el Gateway resuelve ese id a un objeto Host antes de invocar el método de negocio. La composición del Host puede usar `ctx.typert.lookups.configure()` para sobrescribir la política de resolución de una clave de lookup sin cambiar el nombre del parámetro, el campo de cable ni el símbolo de tipo canónico que posee el paquete de negocio.

`@RemoteScope(key)` primero resuelve una identidad a un Context con ámbito mediante `ctx.typert.contexts`, luego obtiene el servicio de ese Context e invoca el método. Se aplica cuando el método en sí depende de la composición con ámbito y no necesita recibir objetos como `Agent` de forma explícita.

Los servicios normalmente extienden `TypertRemoteService` para que el constructor enlace explícitamente la clave de servicio de Cordis y el namespace Remote predeterminado. Un servicio que ya tiene otra clase base puede declarar en su lugar `readonly typertRemote = bindTypertRemote(this, serviceKey)`; ambas formas dejan un enlace público inspeccionable y no dependen de que el compilador inyecte un símbolo en el constructor.

```ts
import type { Agent } from '@deepseek-ai/dsh-agent'
import { TypertRemoteService, Remote, RemoteScope } from '@deepseek-ai/dsh-typert-protocol'
import type { Context } from '@deepseek-ai/cordis'

export interface CreateGoalRequest {
  objective: string
}

export interface CreateGoalResult {
  accepted: boolean
}

export class GoalService extends TypertRemoteService {
  constructor(ctx: Context) {
    super(ctx, 'goals')
  }

  @Remote('create')
  createForClient(
    agent: Agent,
    request: CreateGoalRequest,
    signal: AbortSignal,
  ): CreateGoalResult {
    signal.throwIfAborted()
    return this.create(agent, request)
  }

  @RemoteScope('agent', 'current')
  currentForClient(): CreateGoalResult {
    return { accepted: true }
  }

  private create(_agent: Agent, request: CreateGoalRequest): CreateGoalResult {
    return { accepted: request.objective.length > 0 }
  }
}
```

Los métodos Remote pueden devolver un valor de forma síncrona o devolver una Promise. Para la cancelación cooperativa, el último parámetro de la firma del Host debe ser `signal: AbortSignal` con el tipo global; se registra en el descriptor en lugar de entrar en `args`, mientras que el método de Client generado acepta un `AbortSignal` final opcional.

El Client usa funciones concretas sobre objetos ordinarios, no un Proxy de JavaScript. Las llamadas directas y con ámbito aparecen bajo `ctx.remote.<namespace>` y `agentCtx.remote.<namespace>`. Cada namespace es un Servicio hijo de Cordis con seguimiento registrado como `remote.<namespace>`; el ensamblaje de Client monta las contribuciones mediante `ctx.remote.$mount()`, y el namespace se descarga cuando se retira su último método. Las declaraciones de dependencias pertenecen al llamador real: solo un paquete de negocio que lee `ctx.remote.<namespace>` o `agentCtx.remote.<namespace>` declara tanto `remote` como `remote.<namespace>` en su propio `inject`; los ensamblajes que solo montan contribuciones y los runtimes de nivel superior que no llaman a ese namespace no declaran la dependencia del namespace en nombre del paquete de negocio. Cuando un método `@Remote` tiene exactamente un parámetro de lookup y un `TypertContextMap` con el mismo nombre usa la misma identidad de cable, la firma con ámbito generada omite ese parámetro de identidad. `@RemoteScope` genera solo la interfaz de invocación con ámbito.

```ts ignore-check
import type { SessionId } from '@deepseek-ai/dsh-session/types'
import type { AgentContext } from '@deepseek-ai/dsh-client-runtime/client'
import type { Context } from '@deepseek-ai/cordis'
import type {} from '@deepseek-ai/dsh-api-remotes/client'

export const inject = ['remote', 'remote.goals']

declare const ctx: Context
declare const agentCtx: AgentContext
declare const agentId: SessionId

await ctx.remote.goals.create(agentId, { objective: 'ship it' })
await agentCtx.remote.goals.create({ objective: 'ship it' })
```

Las aplicaciones de Client ensamblan solo `@deepseek-ai/dsh-api-remotes`. Ese paquete importa los subpaths `/remote` de los paquetes de negocio seleccionados como valores de runtime, monta sus contribuciones mediante `ctx.remote.$mount()` y reexporta las fusiones de declaraciones de los mismos archivos. Añadir un paquete Remote de Host es una decisión explícita del responsable de la composición de Client; los componentes de negocio no necesitan cargar por separado el Typert Gateway ni el JS Remote del paquete de negocio.

El ensamblaje `api-remotes` y el contrato de `ctx.remote` son independientes de React; los métodos de Host visibles para cualquier ensamblaje de Client se limitan a los métodos Remote seleccionados en el momento de la generación.

## Responsabilidades de los componentes

| Ubicación | Paquete o entrada | Responsabilidad |
|---|---|---|
| Compartido | `@deepseek-ai/dsh-typert-protocol` | Declara decoradores, enlaces de Gateway, mapas de protocolo extensibles por fusión, descriptores de invocación y tipos de provider; no inicia ningún análisis de TypeScript ni registra servicios de Cordis |
| Compilación | `@deepseek-ai/dsh-typert-generator` | Analiza estrictamente las firmas Remote, el grafo de tipos, los lookups, los Contexts y las ubicaciones en el código fuente desde el `ts.Program` del Host, y luego genera los artefactos de Host y de Host-para-Client |
| Host | `@deepseek-ai/dsh-typert-registry` y Loader | Coloca los descriptores de Host generados, los schemas y los registros de los paquetes de negocio en `ctx.typert`, y mantiene los providers de lookup y de Context |
| Host | `@deepseek-ai/dsh-api-remotes` | Posee la política de identidad de Agent/Session de la aplicación y configura los lookups de Typert correspondientes |
| Host | `@deepseek-ai/dsh-api-gateway` | Proporciona `ctx.typertGateway`, reclama los endpoints Remote, resuelve objetos o Contexts, invoca los servicios de Cordis en vivo y valida los valores de solicitud y de retorno |
| Client | `@deepseek-ai/dsh-api-gateway/client` | Proporciona `ctx.remote` y los Servicios hijos `remote.<namespace>`, monta los descriptores generados como métodos concretos, e inicia, valida y cancela llamadas a través de la Connection |
| Client | `@deepseek-ai/dsh-api-remotes/client` | Selecciona y monta explícitamente las contribuciones `/remote` que permite la aplicación y aporta las fusiones de declaraciones correspondientes al código de negocio |
| Ambos | `@deepseek-ai/dsh-client-connection` | Proporciona el carrier de RPC, la correlación de solicitudes, la frontera de confianza, la cancelación, el sobre de respuesta y el puente HTTP `/api` |

El paquete API Gateway posee el dispatcher de Host y el endpoint Remote de Client como entradas peer, pero las dos compilaciones nunca entran en el mismo `ts.Program`. La entrada de Host no importa la fusión del `Context` de Cordis de Client, y la entrada de Client no importa el servicio del Gateway de Host.

## Pipeline de generación estricto

La compilación raíz ejecuta `build:lib:host`, `build:lib:client` y `build:web` en ese orden. La fase de lib de Host primero ejecuta `tsc -b tsconfig.host.json` y luego `tsdown --env.DSH_BUILD_FACE host`; el grafo normal de Project References de Host compila el generador de Typert, que se ejecuta durante este paso de tsdown con el agregado de Host como única semilla de `ts.Program`. La fase de lib de Client ejecuta después `tsc -b tsconfig.client.json` y `tsdown --env.DSH_BUILD_FACE client`, consumiendo las declaraciones Remote de Client recién generadas y las contribuciones de runtime sin volver a iniciar Typert.

Ambos pasos de tsdown reciben el workspace completo y empaquetan solo el JavaScript que la fase tsc correspondiente emite en `lib/types`. La configuración raíz no escanea los artefactos de Client, no clasifica los nombres de los paquetes ni pasa un filtro mantenido a tsdown; las configuraciones locales de cada paquete devuelven las entradas de la fase actual según `DSH_BUILD_FACE`. Un plugin de Client ordinario produce tanto su entrada de loader de Node como su bundle de navegador durante la fase de Client.

`api-remotes` es el único paquete con caras TypeScript divididas. Su proyecto de Host posee la política de lookups de Agent/Session, mientras que su proyecto de Client depende de las declaraciones `/remote` generadas para los paquetes de negocio durante el tsdown de Host; los agregados raíz y los consumidores directos deben referenciar `api/remotes/tsconfig.host.json` o `api/remotes/tsconfig.client.json` respectivamente. El `clientBundle(..., { hostPhase: true })` del paquete produce su entrada de Host durante el tsdown de Host y deja solo la entrada de navegador para el tsdown de Client. Todos los demás paquetes permanecen registrados en un único agregado.

Cada paquete de negocio que contribuye escribe los archivos generados en su propio directorio `lib/`, no en su directorio de código fuente:

| Archivo | Consumidor | Contenido |
|---|---|---|
| `typert.host.js` | Loader de Host | Reflexión de runtime para la cara de Host, descriptores de invocación estrictos y valores de registro de schema |
| `typert.host.d.ts` | Sistema de tipos de Host | Declaraciones generadas para la cara de Host |
| `typert.remote-client.js` | `api-remotes` | Una `TypertRemoteContribution` montable con descriptores estrictos y codecs de runtime |
| `typert.remote-client.d.ts` | Sistema de tipos de Client | Fusiones de declaraciones para `TypertRemoteNamespaceMap` y `TypertRemoteScopeMap`, más referencias de tipos seguras para Client |
| `typert.remote-client.d.ts.map` | Editor | Mapea las propiedades de método generadas de vuelta a las declaraciones de método Remote del paquete de Host |

Los paquetes de negocio exponen la entrada del Loader de Host a través de `./typert` y la entrada Host-para-Client a través de `./remote`. El generador también valida estas exportaciones de paquete y las listas de archivos publicados; solo genera artefactos para los paquetes de contribución explícitos que proporcionan la entrada correspondiente.

Los nombres de parámetro de las declaraciones Remote de Client provienen de los campos de cable, mientras que los tipos de parámetro y de retorno referencian los tipos seguros para Client exportados por el paquete de negocio original. El mapa de declaraciones resuelve la propiedad generada detrás de `ctx.remote.goals.create` de vuelta al método fuente del Host marcado con `@Remote`, de modo que los editores que admiten mapas de declaraciones pueden navegar desde una llamada de Client hasta la implementación real en lugar de detenerse en el `.d.ts` generado.

El análisis estricto exige que un Remote sea un método de instancia público y no estático con una implementación concreta. El método no puede ser genérico; los parámetros deben ser obligatorios, identificadores simples con nombre y no pueden usar desestructuración, valores por defecto, parámetros rest ni parámetros opcionales. Typert genera schemas estrictos para los tipos ordinarios representables en JSON; los objetos complejos, como las clases de workspace, deben tener una declaración única de `TypertLookupMap`. Los paquetes de lookup y de Context son responsables tanto de las fusiones de declaraciones estáticas como del registro de providers en runtime; si falta cualquiera de los dos lados, la compilación falla o falla la primera llamada que necesita el provider.

## Invocación en runtime

Remote y API Proxy comparten la ruta `/api` de la Connection. El Remote de Client llama a `connection.rpc.call('/api', '<namespace>/<method>', { args }, signal)`; el carrier HTTP lo mapea a `POST /api/<namespace>/<method>`, con una carga útil que contiene solo un objeto `args` con nombre.

La Connection realiza la comprobación de confianza unificada para `/api` antes del puente HTTP y luego despacha dentro del FetchHandler compartido en orden de interceptores. El Typert Gateway reclama únicamente los endpoints de dos segmentos que tienen un descriptor estricto o un marcador SRC activo; las solicitudes no reclamadas caen al API Proxy existente. La Connection es dueña del transporte, de los ids de RPC, de los sobres de respuesta y de la cancelación de solicitudes, mientras que el Gateway es dueño solo del protocolo de datos Remote y del despacho de negocio. Sustituir el carrier de la Connection en el futuro no exige cambios en los descriptores Remote ni en la interfaz de programación de Client.

Para cada llamada, el Gateway resuelve el descriptor y el servicio en vivo desde los registros actuales en lugar de almacenar en caché objetos de negocio. Exige que los campos de `args` coincidan exactamente con el descriptor, valida los valores de cable con codecs, resuelve objetos o receptores mediante los providers de lookup o de Context registrados, invoca el método de servicio al que apunta el enlace y valida el valor de retorno. Un provider ausente, una identidad desconocida, un desajuste de enlace, un argumento ausente o de más, un fallo de schema o un método ausente fallan antes de entrar en el código de negocio o después de salir de él.

El `register()` del provider de lookup aporta tanto la declaración estable como el resolver predeterminado; `configure()` aporta un resolver propiedad de la composición del Host que puede ejecutarse de forma asíncrona y está acotado a la vida de un effect. La configuración puede preceder al montaje del provider; sin provider, la invocación sigue fallando con `lookup-unavailable`, y descargar la configuración restaura la política predeterminada del provider. API Remotes posee la semántica estándar de `agentFor()` para `agent` y `session`: reutiliza un Agent vivo, reanuda automáticamente las sesiones frías ordinarias, deduplica las reanudaciones concurrentes y rechaza las identidades propiedad del enrutado de subagentes; el lookup `session` devuelve la Session de ese Agent. El Web API Proxy aporta sus valores predeterminados de Agent y la configuración de ámbito, y luego consume el mismo resolver para los métodos heredados. Los fallos de reanudación y las barreras de propiedad se propagan sin cambios como errores RPC existentes, en lugar de colapsarse en el error `internal` del Gateway.

Descargar una contribución de Client elimina juntos sus descriptores y sus métodos concretos, aborta sus llamadas en curso y hace que los manejadores de método obsoletos que conserve código externo rechacen llamadas posteriores. Un endpoint estricto retirado en el Host tampoco degrada a inferencia SRC, lo que impide que una descarga en caliente debilite silenciosamente la validación.

## Respaldo de desarrollo SRC

Cuando el Host se inicia desde el código fuente mediante `node --import tsx/esm`, no ejecuta el plugin del compilador de Typert. Los inicializadores de decoradores estándar siguen registrando el nombre del método y el modo de invocación en un `WeakMap` privado del módulo, mientras que `TypertRemoteService` o `bindTypertRemote()` aportan el enlace de servicio explícito; el Gateway puede entonces construir un descriptor temporal más débil sin iniciar un `ts.Program`.

El respaldo SRC analiza los nombres de parámetro simples de la función viva. Cuando un nombre de parámetro coincide con el `parameter` de un lookup registrado, como `agent` o `session`, usa el campo de cable `agentId` o `sessionId` del lookup y resuelve el objeto en el Host; los demás parámetros solo se comprueban como datos sin ciclos, seguros para JSON y sin prototipo especial. `@RemoteScope` usa directamente el campo de cable de un provider de Context de Host registrado. SRC no lee los tipos de TypeScript, no genera schemas de Zod, no infiere parámetros opcionales ni admite desestructuración, valores por defecto, parámetros rest ni nombres de parámetro duplicados.

SRC solo resuelve el despacho de un proceso de Host que se ejecuta desde el código fuente. El Client no descubre decoradores del Host en ejecución, y el Remote de Client se niega a montar descriptores SRC que carecen de codecs estrictos; sus tipos, codecs y valores de registro Remote provienen siempre de los artefactos `lib/typert.remote-client.*` más recientemente generados.

## Modo de desarrollo

El desarrollo web prepara los artefactos actuales de Host, Client y Web con `pnpm run build` y luego ejecuta el Host desde el código fuente y el watcher del plugin de Client en terminales separadas:

```sh
pnpm dsh web
pnpm run dev:web
```

`dsh` inicia el código fuente del Host mediante tsx, de modo que el Host puede usar el respaldo SRC; `dev:web` vigila solo los plugins de Client con una declaración `dsh.client` y reescribe su `lib/client.js`. No analiza los decoradores de Host ni genera DTS de Remote de Client.

Cambiar solo el cuerpo de la implementación de un método Remote sin cambiar su contrato no exige regenerar los archivos de Typert. Tras añadir o quitar un decorador, o cambiar un nombre de exportación, un namespace, un parámetro, un valor de retorno, un lookup, un Context o la firma de cancelación, vuelve a ejecutar la compilación de lib ordenada para que el Host genere el contrato estricto antes de que el Client compile y empaquete la nueva contribución:

```sh
pnpm run build:lib
```

El watcher de Client en ejecución consume estos archivos generados cuando vuelve a empaquetar. Si `pnpm run build:lib:host` ya ha actualizado el contrato de Host, `pnpm run build:lib:client` puede completar el lado de Client; un worktree limpio no puede omitir la fase de Host. Recompilar solo el código fuente del frontend no puede inferir tipos nuevos de los decoradores de Host. `pnpm run typecheck` ejecuta la fase de lib de Host antes del tsc de Client, y las compilaciones de CI y de release usan el mismo orden.

## Límites

Remote atiende solo llamadas a métodos unarios con una solicitud y un resultado. Los flujos de eventos de sesión, la paginación, el reduce incremental, la proyección y los subflujos de entidades requieren un protocolo de datos aparte y un modelo de registro propio; incluso cuando reutilizan la Connection, no deben hacerse pasar por métodos Remote ni entrar en los descriptores de invocación.

Las capas de API se organizan como `remotes → gateway → connection → webserver`. Las capas de BFF y de RPC de Typert viven en `packages/api`; la Connection y el WebServer viven en `packages/client/connection` y `packages/host/webserver`. El API Proxy en `packages/host/apiproxy` atiende los endpoints sin descriptores Remote.

La política de lookups se configura por clave, de modo que todos los parámetros `agent` o `session` comparten el comportamiento de reanudación en frío. Aceptar solo objetos vivos exigiría una política explícita por parámetro o por endpoint, que no existe; el método de negocio no debe adivinar si el objeto proviene de una restauración.
