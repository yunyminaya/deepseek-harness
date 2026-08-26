# Agent Note: Llamadas a métodos dirigidas del Gateway Typert

Status: implemented

[English](2026-08-02-typert-remote-method-calls.md) | [中文](2026-08-02-typert-remote-method-calls.zh.md) | Español

## Problema

El API Proxy del Host gestiona las llamadas directas a métodos, las interacciones con estado y los flujos de eventos de Session. Estas responsabilidades tienen ciclos de vida, semánticas de enrutamiento e interfaces de programación de cliente distintos. Seguir exportando todas las operaciones de negocio a través de un único paquete acoplaría los Services de negocio, los protocolos de transporte, las máquinas de estado y los tipos de cliente.

Esta decisión cubre solo las llamadas a métodos dirigidas en las que una solicitud produce un resultado. Las interacciones con estado como Permission y Approval, así como los flujos de eventos de Session, siguen siendo diseños separados.

El contrato de una llamada directa a un método pertenece al Service de negocio que lo implementa. Los desarrolladores de negocio declaran solo qué métodos son invocables remotamente, sin mantener además una interfaz de API central, una tabla de enrutamiento, una tabla de conversión de parámetros, un stub de cliente y un schema Zod.

El Host y el cliente de navegador usan Programas de TypeScript separados porque cada lado aumenta el tipo `Context` de Cordis de forma distinta. Una proyección Remote no debe importar las declaraciones completas del Host en un consumidor ni depender de tipos específicos del navegador. Si el TUI reutiliza más adelante esta interfaz de programación, debe ver igualmente solo los métodos marcados como Remote. La integración del TUI queda fuera del alcance actual, pero el límite de implementación debe preservar esta reutilización isomórfica.

## Decisión

Un Service de negocio extiende `TypertRemoteService` y declara los métodos invocables con `@Remote` o `@RemoteScope()`. Un Service que ya tiene otra clase base puede exponer en su lugar el mismo binding mediante `bindTypertRemote()`. Typert genera el artefacto de reflexión local del Host y una proyección Remote de consumo independiente de la plataforma a partir del Programa del Host. El Programa del cliente sigue generando su propio artefacto de reflexión local de forma independiente.

La proyección Remote de consumo contiene archivos `.d.ts`, `.d.ts.map` y `.js`. El `.d.ts` expone solo los métodos marcados con un decorador Remote y remite a los símbolos de tipo públicos únicos del paquete de negocio. El `.d.ts.map` lleva los métodos de la API de consumo de vuelta a sus implementaciones de métodos de negocio del Host. El `.js` transporta la información de endpoint, parámetros, Context y Zod del mismo contrato. En la capa de assembly, el cliente de navegador monta las contribuciones Remote JS necesarias sobre el Service Remote de cliente. La proyección y la abstracción Remote siguen siendo independientes de la plataforma para que un TUI futuro pueda reutilizarlas.

`@deepseek-ai/dsh-api-gateway`, ubicado en `packages/api/gateway`, ofrece dos caras simétricas: su entrada por defecto proporciona el `ctx.typertGateway` del Host, mientras que su entrada `/client` proporciona el `ctx.remote` del lado de consumo. Cada lado consume un `InvocationDescriptor` generado localmente a partir del mismo modelo; los descriptores no se envían por el wire. El protocolo de datos Remote corre sobre el canal RPC `/api` compartido de Connection. La interfaz de llamada de negocio no cambia cuando Connection migra de HTTP a WebSocket.

`@deepseek-ai/dsh-api-remotes`, ubicado en `packages/api/remotes`, es la capa BFF por encima del Gateway. Su entrada del Host es dueña de la resolución de identidad Agent/Session y de la configuración de lookup de Typert; su entrada `/client` selecciona las contribuciones Remote generadas que expone la aplicación. La entrada de cliente consume el contrato compartido `TypertClientRemote` a través de Cordis en lugar de importar la implementación concreta del Gateway.

## Componentes y services de Cordis

| Componente | Service de Cordis | Responsabilidad |
|---|---|---|
| `@deepseek-ai/dsh-typert-protocol` | Declara solo el protocolo mínimo `ctx.typert` | `TypertRemoteService`, decoradores, binding de respaldo, descriptores, lookup/Context y el mapa Remote; sin dependencia del compilador, de Zod, de Connection ni del navegador |
| Registro de Typert | `ctx.typert` | Almacena por separado la reflexión del entorno actual, las contribuciones Remote importadas, los providers de lookup y los providers de Context |
| Generador/loader de Typert | Ningún Service de negocio nuevo | Genera tres tipos de artefactos `lib` a partir de los Programas del Host/Cliente y registra los artefactos del entorno actual con `ctx.typert` |
| Cara del Host del API Gateway | `ctx.typertGateway` | Asocia las definiciones del Host con Services vivos, decodifica parámetros, resuelve receptores, invoca métodos y codifica resultados |
| Connection | `ctx.connection` | Es dueño exclusivo del servidor HTTP/WebSocket futuro, de la ruta `/api` compartida, del envelope RPC, del rpcId, de la serialización, de la confianza, del transporte de errores, de la intercepción Typert y del fallback del API Proxy heredado |
| Cara de cliente del API Gateway | `ctx.remote`, `ctx.remote.<namespace>` | Monta las contribuciones Remote, materializa cada espacio de nombres como un Service hijo `remote.<namespace>` trazado y delega las llamadas canónicas en `ctx.connection.rpc` |
| API Remotes | Ningún Service nuevo | Es dueño de la política de lookup de Agent/Session del Host y sirve como única fachada de negocio del cliente, seleccionando y montando las contribuciones `/remote` mientras expone las declaraciones de API seleccionadas |
| Paquetes dueños de Agent/Session | Services de dominio existentes | Aportan tanto las fusiones de interfaces estáticas como los providers de lookup/Context en runtime |
| Paquetes de negocio como Goal | Services de negocio existentes | Declaran solo bindings, métodos Remote y DTO canónicos, y exportan el subpath `/remote` generado |

El Host Gateway no depende de implementaciones concretas de `ctx.agents`, `ctx.sessions`, `ctx.goals` ni `ctx.webServer`. El Remote de cliente no entiende el carrier físico, y Connection no entiende Goal, Agent, lookup, `InvocationDescriptor` ni los espacios de nombres Remote.

## Declaraciones de negocio

Las llamadas directas ordinarias usan `@Remote`. Cuando los parámetros y el resultado de un método existente ya son el contrato Remote previsto, decora ese método directamente sin renombrarlo. Añade un adaptador `remoteExport*` solo cuando el contrato wire necesite una forma de solicitud o de resultado distinta, y usa el argumento del decorador para declarar su nombre corto de API. Un método declara explícitamente cada objeto de negocio necesario en una posición de parámetro de nivel superior:

```text
export class GoalService extends TypertRemoteService {
  constructor(ctx: Context) {
    super(ctx, 'goals')
  }

  create(agent: Agent, request: CreateGoalRequest): GoalView {
    // Existing business method remains unchanged.
  }

  @Remote('create')
  remoteExportCreate(agent: Agent, request: CreateGoalRequest): CreateGoalResult {
    const view = this.create(agent, request)
    return { ref: { id: view.id, revision: view.revision } }
  }
}
```

`goals` es la clave de Service de Cordis explícita que se pasa a `super()` y es el espacio de nombres wire por defecto. Pasa una opción `namespace` como tercer argumento solo cuando el espacio de nombres del protocolo necesite de verdad diferir de la clave de Service.

Usa `@RemoteScope()` cuando el receptor del Service deba resolverse dentro de un tipo aislado de Context. La identidad de Scope no entra en los parámetros del método de negocio:

```text
export class ScopedGoalService extends TypertRemoteService {
  constructor(ctx: Context) {
    super(ctx, 'goals')
  }

  @RemoteScope('agent', 'create')
  remoteExportCreate(request: CreateGoalRequest): Promise<CreateGoalResult> {
    // Runs against the goals service resolved from the Agent Context.
  }
}
```

Un endpoint selecciona exactamente un modo de invocación. Un flujo que necesita un parámetro `Agent` explícito usa `@Remote`. Un flujo que primero cambia a un Agent Context y luego resuelve un receptor con ámbito usa `@RemoteScope('agent')`. Typert no infiere ninguno de los dos modos a partir del cuerpo del método ni de un parámetro ausente.

Los paquetes de negocio dependen solo del ligero `@deepseek-ai/dsh-typert-protocol`. Proporciona `TypertRemoteService` y los protocolos de declaración para decoradores, el binding de respaldo, el lookup, el Remote Scope y los descriptores, sin depender del compilador de TypeScript, de Zod, de HTTP ni del runtime del cliente.

Un método que admite cancelación de forma cooperativa declara `signal: AbortSignal` como su último parámetro del Host. Este parámetro reservado no es un valor de negocio, un lookup ni un campo JSON. El método de consumo generado lo expone como último parámetro opcional, de modo que las llamadas ordinarias permanecen sin cambios mientras los llamadores que poseen la cancelación pueden pasar una señal.

## Decoradores y la faceta explícita del Gateway

Un decorador solo afirma que un método participa en el contrato Remote. No realiza reflexión de tipos en runtime ni inyecta ningún símbolo oculto en un constructor de Service. Los argumentos de `@Remote('create')` y `@RemoteScope('agent', 'create')` son nombres de métodos externos; el miembro decorado puede ser el propio método de negocio o un adaptador como `remoteExportCreate`. El nombre del miembro se convierte en el nombre del método externo solo cuando no se proporciona un alias. Heredar de `TypertRemoteService` es la declaración explícita normal de que un Service se ha unido al Gateway; su campo público de solo lectura `typertGateway` mantiene el binding visible en la instancia en runtime.

En modo SRC, el decorador puede registrar el prototipo, el nombre del método y el modo de invocación en un `WeakMap` interno de `dsh-typert-protocol`. No escribe propiedades personalizadas en una instancia de Service, prototipo, constructor ni función de método.

En modo LIB, el compilador de Typert realiza el descubrimiento estricto de métodos, la resolución de tipos y la generación de descriptores. Acepta una clave de Service literal en la llamada directa a `super()` de `TypertRemoteService` o el binding de respaldo explícito; la generación ni reescribe el código fuente de negocio ni inyecta metadatos de registro ocultos.

## Registro de lookup y Remote Scope

El Gateway no tiene ramas integradas para Agent, Session ni otros objetos de negocio. Cada paquete dueño de objetos proporciona tanto una declaración estática como un provider en runtime:

```text
declare module '@deepseek-ai/dsh-typert-protocol' {
  interface TypertLookupMap {
    agent: TypertLookup<Agent, SessionId>
  }
}

ctx.typert.lookups.register('agent', {
  parameter: 'agent',
  wire: 'agentId',
  resolve: sessionId => resolveAgent(sessionId),
})
```

La declaración estática le dice a Typert que `Agent` se corresponde con `SessionId` en el wire. El provider en runtime resuelve un `agentId` de una solicitud al objeto `Agent` actualmente vivo. Si falta cualquiera de los dos lados, la build LIB o el primer registro en runtime resoluble fallan de inmediato.

Los objetos de lookup como Agent y Session solo pueden ocupar cada uno una posición de parámetro de nivel superior. Una solicitud JSON ordinaria puede pasarse como otro parámetro completo, pero este diseño no admite `request.agent`, la desestructuración de objetos, los arrays de objetos, los lookups anidados ni la búsqueda de IDs en estructuras complejas arbitrarias.

Remote Scope usa un mapa fusionable separado y un provider de Context. El paquete de Agent registra un provider `agent` que localiza el Agent Context a partir de su identidad wire y resuelve desde ese Context la clave de Service que nombra el descriptor. El Gateway no conoce la estructura interna de un Agent Context.

El cliente también registra un binder de Context `agent`. El binder solo recupera un `SessionId` del Context en el que se produce una llamada; no enumera Scopes ni copia métodos en cada uno. Un tracker de Services de Cordis vuelve a vincular automáticamente un espacio de nombres con ámbito al Agent Context actual.

## InvocationDescriptor

Typert, el parser SRC permisivo, el Host Gateway y el Remote de cliente intercambian una única descripción canónica:

```text
InvocationDescriptor {
  id: '@deepseek-ai/dsh-goal#goals/create'
  service: 'goals'
  namespace: 'goals'
  method: 'create'
  implementation: 'remoteExportCreate'
  invocation: direct | { context: 'agent', wire: 'agentId' }
  scope?: { context: 'agent', wire: 'agentId' }
  parameters: [
    { name, wire, source: json | lookup, lookup?, codec }
  ]
  cancellation?: { parameter: 'signal' }
  result: codec
  sourceLocation
}
```

`method` es el nombre corto externo que usan el endpoint y el Remote de cliente; `implementation` es el nombre real del miembro en el receptor del Host. `implementation` puede omitirse cuando ambos nombres coinciden. Un descriptor `direct` conserva la instancia original del Service como receptor. Un descriptor de Context primero usa el provider de Context correspondiente para encontrar el Context con ámbito y luego resuelve el receptor por la clave de Service del descriptor.

El generador estricto escribe `scope` solo cuando un método directo tiene exactamente un parámetro de lookup, existe una declaración `TypertContextMap` con el mismo nombre y ambos usan el mismo símbolo de tipo wire. `scope.wire` debe identificar ese parámetro de lookup. Declara que un consumidor puede rellenar este parámetro desde el Context en el que se produce la llamada, sin cambiar el receptor ni el endpoint del Host. No se genera ninguna proyección con ámbito cuando hay varios lookups, no hay declaración de Context o los tipos wire no coinciden; un desajuste de tipos es un error de build.

El orden de los parámetros proviene de la firma del método. Los campos HTTP provienen de los nombres de los parámetros o de las declaraciones de lookup. Un descriptor de cancelación reserva solo la posición final `signal` y la mantiene fuera de los `args` con nombre; Connection o un llamador directo del Gateway suministra la señal real. El Gateway no infiere campos opcionales, tipos de Context, tipos de lookup ni argumentos ausentes a partir del contenido de la solicitud, y no sintetiza valores por defecto de negocio.

Un codec LIB contiene un schema Zod y un `typeSymbol` canónico que consiste en «paquete + subpath público + nombre de exportación». Un codec SRC se marca solo como `src-json`. Cuando el Host y el consumidor se ejecutan en realms de JavaScript distintos, cada uno mantiene sus propias instancias de Zod, pero ambos conjuntos se generan a partir del mismo modelo de Typert y de las mismas claves de símbolo.

Los descriptores existen solo en el registro local de cada lado. El wire transporta únicamente el canal `/api`, el endpoint y el payload `{ args }`. El Host usa su descriptor para decodificar e invocar el método, mientras que el cliente usa su descriptor correspondiente para codificar los argumentos y validar el resultado.

## Registro en runtime de Typert

```text
ctx.typert.local     当前进程自己的 Host 或 Client reflection
ctx.typert.remotes   消费端显式 mount 的对端 Remote contribution
ctx.typert.lookups   wire ID 到 Host 对象的 provider 与组合策略
ctx.typert.contexts  Host Context resolver 与 Client Context binder
```

Cada registro devuelve un disposer propiedad del fiber de Cordis del llamador. El montaje de contribuciones del cliente registra el conjunto de descriptores y los métodos concretos como una única operación con propietario. El Host Gateway guarda en caché solo el conjunto de nombres de endpoints propiedad de SRC y lo descarta cada vez que cambia el conjunto de Services de Cordis; no retiene ningún descriptor, Service ni provider. La invocación resuelve todos los objetos vivos a partir del estado actual, de modo que eliminar una definición estricta, un Service o un provider hace que la llamada correspondiente deje de estar disponible sin dejar un objeto vivo obsoleto.

El registro de lookup retiene la declaración wire estable después de que su resolver en vivo se descargue. El parsing SRC sigue clasificando el parámetro como lookup, mientras que la invocación falla con `lookup-unavailable`; nunca reclasifica el ID entrante como un objeto JSON de negocio ordinario. Volver a registrar la misma clave con símbolos de parámetro, wire o tipo canónico distintos falla durante toda la vida de ese Typert Service.

Los paquetes de objetos de negocio y de Context con ámbito son dueños de las declaraciones estables y de los resolvers por defecto mediante `lookups.register()` y `contexts.registerHost()`; la composición del Host suministra políticas asíncronas con ámbito de effect mediante `lookups.configure()` y `contexts.configureHost()`. La configuración puede preceder al registro del provider, pero por sí sola no hace que una identidad esté disponible sin un provider en vivo; descargar la configuración restaura el resolver por defecto del provider. API Remotes crea el resolver compartido `agentFor()` para los lookups `agent` y `session`, y el Host Context `agent`: los Agents vivos se reutilizan, las sesiones frías ordinarias se reanudan automáticamente, las reanudaciones concurrentes se deduplican por ID de Session, y la valla de propiedad de subagentes devuelve el `agent-busy` existente. El Web API Proxy estándar suministra sus valores por defecto de Agent y su configuración de scope, y consume ese resolver para los métodos heredados. El lookup `session` devuelve la Session del Agent resuelto, mientras que el Host Context `agent` devuelve su Context, de modo que las tres proyecciones comparten un único ciclo de vida de reanudación.

La entrada raíz del registro para el Host tiene la fusión completa de interfaces de `TypertRegistryContract`. La implementación del registro compartida por Host y cliente vive en un módulo separado sin declaraciones de entorno. La entrada `/client` del registro importa solo esa implementación compartida y no pasa por la entrada raíz del Host, de modo que no puede llevar declaraciones de Cordis del Host al Programa del cliente.

## Tipos canónicos, símbolos y Zod

La DTS de cliente Remote no copia los DTO de negocio ni vuelve a declarar tipos sombra estructuralmente idénticos. Importa los símbolos originales solo desde subpaths públicos de solo tipos que no arrastran las fusiones de Cordis del Host:

```text
import type { SessionId } from '@deepseek-ai/dsh-session/types'
import type { CreateGoalRequest, CreateGoalResult } from '@deepseek-ai/dsh-goal/types'
```

En consecuencia, `SessionId`, el ID wire de Agent, la solicitud y el resultado remiten todos a la misma declaración de TypeScript en el Host y en el cliente de navegador. Un TUI futuro puede reutilizarlos sin un segundo conjunto de tipos. Ir a la definición, los renombrados y Buscar referencias de un DTO vuelven a la única ubicación de origen del tipo de negocio en lugar de detenerse en una copia dentro de un archivo generado.

Los propios métodos Remote usan la navegación por mapa de declaraciones. Typert ancla `InvocationModel.location` al token del nombre del método del Host decorado y emite un segmento de source map en la propiedad correspondiente de la interfaz de espacio de nombres. Para un endpoint respaldado por un adaptador, después de que el editor de TypeScript resuelva `ctx.remote.models.list` hasta su declaración generada, `typert.remote-client.d.ts.map` lo lleva al punto de entrada `remoteExportList` del Service del Host. Ese punto de entrada llama explícitamente al método `list()` existente y sin renombrar; el mapa no identifica erróneamente el decorador, la clase ni la firma completa como definición del método.

Typert genera un codec Zod de wire para la misma clave de símbolo. El Host Gateway lo usa para validar la entrada y codificar los resultados, mientras que el Remote de cliente lo usa para codificar los argumentos y validar las respuestas. Si un tipo complejo no puede producir un codec estricto, la build LIB falla en lugar de degradarse a `unknown` o a JSON sin verificar.

Los tipos de negocio con nombre a los que remiten los métodos Remote deben exportarse desde subpaths públicos de solo tipos. Si la única entrada alcanzable también importa Services del Host, fusiones de `Context` de Cordis o implementaciones solo del Host, la build falla y exige que el paquete de negocio proporcione una entrada de tipos segura. Las primitivas, los literales y las composiciones simples soportadas explícitamente por Typert no necesitan nombres adicionales.

Un parámetro de lookup no expone la clase `Agent` a los consumidores. La proyección Remote remite al tipo de ID canónico de la declaración de lookup, como `SessionId`, mientras que el Host sigue resolviendo los objetos mediante el símbolo canónico de la clase `Agent`.

## Tres tipos de artefactos y dos Programas de TypeScript

El Host y el cliente siguen usando solo dos Programas de TypeScript independientes, pero Typert genera tres tipos de artefactos semánticamente distintos:

```text
Host Program
├─ typert.host.js / typert.host.d.ts
│  Host 自身的 Service、Event、Object、schema 和 inbound Gateway 信息
└─ typert.remote-client.js / typert.remote-client.d.ts / typert.remote-client.d.ts.map
   Host Remote 对任意消费环境的 wire 投影

Client Program
└─ typert.client.js / typert.client.d.ts
   Client 自身的 Service、Event、Object 和 schema 信息
```

`remote-client` es el segundo emisor del Programa del Host, no un tercer Programa ni la cara local del cliente. No contiene ninguna fusión de Cordis del Host, clase de Service, clase de Context ni código de implementación, y no entra en el registro de reflexión local del Host.

La build lib del Host realiza el análisis estricto del Host y emite tanto los artefactos locales del Host como los de consumo Remote. La lib del cliente consume entonces la DTS Remote. El orden completo es:

```text
Host lib build
→ 生成 typert.host.{js,d.ts}
→ 生成各业务包 lib/typert.remote-client.{js,d.ts,d.ts.map}
→ 完成 Client lib 和 typert.client 产物
→ Vite 构建 Web
```

El `build` de nivel superior existente sigue ejecutando `build:lib` antes que `build:web`, pero `build:lib` debe completar los artefactos del Host y Remote antes de iniciar la compilación de TypeScript del cliente. Una build limpia no debe depender de archivos `.d.ts` obsoletos de una build anterior.

Las verificaciones del repositorio respaldadas por el compilador que resuelven la superficie de consumo tienen el mismo prerrequisito incluso cuando sus entradas principales son archivos fuente. Los comandos públicos `typecheck`, `lint` y `doc-typecheck` ejecutan primero la pasada del contrato del Host. El planificador de verificaciones puede usar sus variantes `*:contracts-ready` solo después de una dependencia explícita del contrato Typert o de la build completa, de modo que los carriles en paralelo ni leen declaraciones ausentes ni ejecutan generadores concurrentes sobre las mismas salidas.

## La entrada de paquete `/remote`

Todo paquete de negocio que proporciona métodos Remote exporta un subpath `/remote` generado:

```text
"./remote": {
  "types": "./lib/typert.remote-client.d.ts",
  "default": "./lib/typert.remote-client.js"
}
```

El código de consumo selecciona una capacidad a través del propio paquete de negocio:

```text
import goalsRemote from '@deepseek-ai/dsh-goal/remote'
```

Este import lleva el aumento del mapa `.d.ts` al proyecto de TypeScript actual mientras suministra el descriptor JS del mismo contrato como valor al runtime. Un paquete de negocio que no se importa no extiende los tipos de Remote API del proyecto actual.

Los archivos publicados del paquete de negocio deben incluir `lib/typert.remote-client.d.ts.map`. La DTS generada remite a su mapa adyacente con `//# sourceMappingURL=typert.remote-client.d.ts.map`; el source del mapa apunta desde `lib` al código fuente de negocio mediante una ruta relativa como `../src/index.ts`. La exportación `/remote` no lista el mapa por separado; el campo `files` del paquete lo publica. Ese destino es una ruta de tiempo de desarrollo: un consumidor del workspace lo resuelve a través del enlace del paquete, de modo que el payload publicado sigue excluyendo `src` y un mapa publicado simplemente no resuelve nada.

El código que solo necesita tipos estáticos puede usar `import type {} from '@deepseek-ai/dsh-goal/remote'`. Este import se borra en runtime, no carga JS y no puede disparar registros en runtime. Un entorno que realiza llamadas reales debe pasar la contribución de un import de valor normal al Service Remote de cliente.

La resolución en el workspace de `/remote` debe apuntar explícitamente a los artefactos `lib` generados y no debe permitir que una regla general de rutas de paquete a `src` la redirija al código fuente del Host. Los imports de negocio ordinarios pueden seguir resolviendo a SRC o LIB según las reglas existentes de cada entorno.

## Tipos estrictos de la API de consumo

La DTS Remote extiende el mapa plano de endpoints, la interfaz directa de espacio de nombres, el mapa de espacios de nombres y el mapa con ámbito sin aumentar el `Context` global de Cordis:

```text
interface TypertRemoteNamespace$676f616c73 {
  create: (
    agentId: SessionId,
    request: CreateGoalRequest,
    signal?: AbortSignal,
  ) => Promise<CreateGoalResult>
}

interface TypertRemoteMap {
  'goals/create': (
    agentId: SessionId,
    request: CreateGoalRequest,
    signal?: AbortSignal,
  ) => Promise<CreateGoalResult>
}

interface TypertRemoteNamespaceMap {
  goals: TypertRemoteNamespace$676f616c73
}

interface TypertRemoteScopeMap {
  'agent:goals/create': (
    request: CreateGoalRequest,
    signal?: AbortSignal,
  ) => Promise<CreateGoalResult>
}
```

`TypertRemoteMap` conserva las firmas canónicas de los endpoints para el tipado y la reflexión del protocolo. El tipo Remote raíz lee `TypertRemoteNamespaceMap` directamente en lugar de derivar los métodos de forma indirecta a través de un tipo mapeado con claves remapeadas; el TypeScript Language Service no puede navegar de forma fiable por esas propiedades indirectas a través de un mapa de declaraciones. Un nombre de interfaz de espacio de nombres codifica los bytes UTF-8 del espacio de nombres como hexadecimal, de modo que `goals` se convierte de forma determinista en `TypertRemoteNamespace$676f616c73`. Paquetes distintos generan el mismo nombre de interfaz para el mismo espacio de nombres y usan el aumento de módulos para fusionar sus métodos, mientras que `TypertRemoteNamespaceMap.goals` remite siempre a ese único tipo.
Typert proyecta `TypertRemoteScopeMap` sobre un tipo Scope dedicado según su clave de Context. La interfaz de programación final queda así:

```text
ctx.remote.goals.create(agentId, request)
agentCtx.remote.goals.create(request)
```

El Agent Scope suministra su propio `SessionId` automáticamente. Un método `@Remote` con un lookup `agent` puede por tanto generar tanto la firma de consumo raíz como la con ámbito. Un método `@RemoteScope('agent')` también omite una identidad de Scope separada, pero genera solo la firma con ámbito. El `Context` raíz expone los espacios de nombres directos a través de `ctx.remote`, mientras que `AgentContext.remote` interseca esa superficie directa con la superficie con ámbito. Un TUI futuro debe preservar la misma distinción.

`TypertClientRemote` sigue siendo independiente de la plataforma, y el cliente de navegador lo expone como `ctx.remote`. Si un TUI futuro reutiliza este tipo, debe acceder a él igualmente a través de un objeto Remote dedicado y del Agent Scope en lugar de tratar el `Context` del Host como una colección de Services más amplia. Los métodos públicos de Service sin marcadores Remote no entran en los mapas Remote.

## Typert de cliente y la cara de cliente del API Gateway

Typert en un entorno de consumo mantiene tanto la información local como la información Remote importada de otros entornos, pero las almacena en registros separados:

```text
Typert.local    当前环境自己的反射模型
Typert.remotes  已导入的 Remote contribution
```

`@deepseek-ai/dsh-api-remotes/client` carga de forma central las contribuciones Remote necesarias:

```text
import goalsRemote from '@deepseek-ai/dsh-goal/remote'
import sessionsRemote from '@deepseek-ai/dsh-session/remote'

await ctx.remote.$mount(goalsRemote)
await ctx.remote.$mount(sessionsRemote)
```

Los paquetes de negocio del cliente dependen solo de `@deepseek-ai/dsh-api-remotes/client`, no directamente del API Gateway ni de la entrada en runtime de cada `/remote` de negocio. API Remotes consume el contrato compartido `TypertClientRemote` y el service `ctx.remote` de Cordis, y luego reexporta las declaraciones para que el mapa Remote seleccionado llegue a la compilación de negocio. Añadir o quitar una capacidad completa del cliente cambia solo este punto de assembly.

`ctx.remote.$mount()` registra una contribución con `Typert.remotes`, instala sus Services de espacio de nombres y sus métodos concretos, y se resuelve solo cuando están listos. Su disposer es propiedad del fiber de Cordis que llamó al método. Los endpoints duplicados, los modos de invocación en conflicto para el mismo espacio de nombres y método, o los conflictos entre un descriptor y una identidad de tipo existente fallan de inmediato.

El Service Remote de cliente materializa cada descriptor `@Remote` como una función real sobre un Service hijo `remote.<namespace>`. La función construye los `args` con nombre en el orden de parámetros del descriptor, aplica el codec estricto del cliente y luego llama a `ctx.connection.rpc.call('/api', endpoint, { args }, signal)`. Para un descriptor consciente de la cancelación, la función generada acepta una señal final opcional y la combina con el ciclo de vida del montaje de la contribución; desmontar cancela por tanto todas las llamadas de carrier en vuelo, mientras que un llamador puede cancelar una llamada de forma independiente.

Ni un descriptor directo con `scope` ni un descriptor `@RemoteScope` copian funciones en cada Agent Scope. El Service Remote de cliente crea un Service hijo de Cordis por espacio de nombres, registrado como `remote.<namespace>`, y materializa sobre él las variantes directa y con ámbito. Acceder a un método a través de `agentCtx.remote.goals` captura el Agent Context actual antes de devolver el handle invocable. El método pide entonces al binder de Context correspondiente la identidad de ese Context. Una proyección directa con ámbito sustituye esta identidad en la posición de lookup que nombra `scope.wire`; un descriptor Remote Scope escribe la identidad en el campo wire separado del receptor. Ambos emiten el mismo tipo de llamada `/api`.

```text
root ctx.remote.goals.create(agentId, request)
  → direct descriptor
  → ctx.connection.rpc.call('/api', 'goals/create', { args })

agentCtx.remote.goals.create(request)
  → remote.goals accessor 捕获 agent Context
  → agent binder 从 caller Context 取得 agentId
  → 用 agentId 补入同一 direct descriptor 的 lookup 参数
  → ctx.connection.rpc.call('/api', 'goals/create', { args })
```

El `Context` raíz fusiona solo la superficie directa de `TypertClientRemote`. `AgentContext` reemplaza esa propiedad con la intersección de `TypertClientRemote` y `TypertRemoteScopeApi<'agent'>`, de modo que los métodos solo con ámbito siguen sin estar disponibles desde el código raíz. Si un llamador se salta el sistema de tipos y llama dinámicamente a un método solo con ámbito desde Root, el binder informa de un error explícito. Si el cliente ya tiene un service de Cordis llamado `remote.<namespace>`, o dos contribuciones reclaman el mismo espacio de nombres y método de forma incompatible, el montaje falla en lugar de sobrescribir el service existente.

El JS Remote generado contiene solo descriptores, claves de símbolo y codecs; no empaqueta implementaciones de Services del Host. El Service Remote de cliente crea funciones reales a partir de esos datos, de modo que el runtime no depende de un Proxy de JavaScript. Un Proxy sigue siendo una opción de implementación, pero no es una fuente de tipos ni de reflexión.

## Restricciones de isomorfismo entre entornos

Remote API es una capacidad de consumo, no un sinónimo de Browser API. El runtime publicado implementa el montaje de contribuciones del cliente de navegador, las llamadas RPC de Connection y la asociación con el Agent Scope.

La DTS Remote, el JS Remote, `TypertClientRemote`, `InvocationDescriptor`, el protocolo de datos RPC Remote y los binders de Context no deben depender del DOM, de los loaders de módulos del navegador ni de HTTP. A través de Connection, el cliente de navegador codifica los métodos materializados desde descriptores como llamadas RPC `/api`.

Un TUI futuro puede unirse a la misma abstracción de llamadas sin cambiar los decoradores de negocio, los mapas Remote ni la forma de las llamadas de API. La API visible para el TUI debe seguir generándose exclusivamente a partir de `@Remote` y `@RemoteScope`; compartir un proceso con el Host no debe permitirle saltarse las restricciones Remote y exponer métodos de Service directamente.

El montaje en runtime del TUI, los carriers, la asociación con el Agent Scope y el cableado de arranque SRC quedan diferidos fuera de esta decisión.

El Web ya depende de artefactos de build como `lib/client.js`, así que requiere un `build:lib` completo antes del arranque. Cuando el contrato Remote del Host cambia, los desarrolladores reconstruyen la lib y luego arrancan o reinician el Web. La observación incremental del contrato Remote no está implementada.

## Modos de funcionamiento SRC y LIB

SRC admite el arranque local desde el código fuente. Los registros `WeakMap` creados por `@Remote` y `@RemoteScope()` proporcionan los nombres de los métodos y los modos de invocación. En runtime, el sistema lee los nombres ordenados de los parámetros de la firma de la función de JavaScript y los combina con los providers de lookup/Context registrados para producir un descriptor permisivo.

Por ejemplo, `@Remote('create') remoteExportCreate(agent, request, signal)` se resuelve al método externo `create`, al miembro de implementación `remoteExportCreate`, a dos parámetros de negocio de nivel superior y a un punto de inyección de cancelación. El registro de lookup reescribe `agent` como el campo wire `agentId`, `request` se pasa como un parámetro JSON con el mismo nombre, y la `signal` final queda fuera del payload. SRC no arranca un `ts.Program`, no usa un hook de preload ni de loader, no genera ni reescribe código fuente, ni inspecciona la estructura interna de un objeto JSON ordinario.

Una firma que SRC no puede resolver sin ambigüedad falla en la primera invocación que resuelve su descriptor; el montaje del Service registra solo el marcador del decorador y no inspecciona la firma de JavaScript. SRC no adivina la desestructuración de objetos, la ambigüedad causada por parámetros por defecto, los parámetros rest, los lookups anidados ni los tipos complejos.

LIB admite CI, releases y la build Web prerrequisito. Typert escanea el proyecto completo del Host y comprueba los decoradores Remote, los bindings explícitos, las claves de Service, los conflictos de endpoints, las declaraciones de lookup/Context, la alcanzabilidad de los símbolos públicos, los codecs JSON, los codecs de resultado y que un parámetro final reservado `signal` tenga el tipo global `AbortSignal`, y luego genera descriptores estrictos.

En runtime, LIB solo carga definiciones desde `lib`; no arranca el compilador de TypeScript. La asociación posterior de Services, el lookup, la resolución de Context, la invocación y la codificación de la respuesta en el Host Gateway no dependen de si un descriptor provino del parsing SRC permisivo o de la generación LIB estricta.

CI y los releases usan LIB. Trasladar toda la cobertura del repositorio a LIB es trabajo de seguimiento aparte y no bloquea esta implementación de llamadas directas a métodos.

## Resolución en el Host Gateway

El Host Gateway registra un interceptor `/api` con Connection y no mantiene un segundo registro de endpoints. Su comprobador de titularidad comprueba primero el registro local actual de Typert y luego consulta un conjunto consciente de la invalidación, poblado al escanear los Services de Cordis actuales en busca de bindings `typertGateway` y marcadores Remote de SRC. Un cambio en los Services de Cordis descarta el conjunto, de modo que las definiciones de Typert y los Services de negocio pueden llegar en cualquier orden sin obligar al tráfico `/api` heredado a reescanear cada Service en cada solicitud ni permitir que rutas de solicitud arbitrarias hagan crecer la caché.

La invocación vuelve a resolver el descriptor, el receptor, los providers de lookup y el provider de Context a partir del estado actual. Un descriptor estricto actual tiene prioridad sobre SRC. Después de que aparezca un endpoint estricto, `TypertLocalRegistry.hasSeen()` lo mantiene con propietario cuando ese descriptor se retira y prohíbe el fallback a SRC durante el resto de la vida del registro; volver a registrar el descriptor estricto restaura las llamadas. Eliminar un Service o provider hace que la invocación falle explícitamente, y el Gateway ni retiene objetos inválidos ni invoca un método con un ID de lookup en bruto.

Una llamada `@Remote` ordinaria conserva la instancia original del Service como receptor. Cuando los lookups tienen éxito, el Gateway llama al miembro identificado por `implementation ?? method` con los parámetros en el orden del descriptor, seguidos de la señal del carrier cuando el descriptor declara cancelación.

Una llamada `@RemoteScope('agent')` primero pide al provider de Agent Context que resuelva la identidad wire, luego lee la clave de Service del descriptor desde ese Context e invoca al receptor con ámbito. El método de negocio no recibe ni un parámetro de Context oculto ni un ID de Agent.

```text
ctx.typertGateway.invoke({ namespace, method, args, signal })
→ 查找本地 InvocationDescriptor 与 live receiver
→ 按参数 descriptor 读取具名 wire 字段
→ codec 解码普通值或 lookup ID
→ lookup provider 把 ID 解析为活对象
→ direct 使用原 Service；context 先解析 scoped Context 和 Service
→ cancellation descriptor 存在时把 signal 追加到业务参数末尾
→ Reflect.apply(receiver[implementation ?? method], receiver, orderedArgs)
→ result codec 编码业务结果
```

`ctx.typertGateway.invoke()` es el punto de entrada del Host independiente del carrier. No crea ni un rpcId, ni un envelope RPC, ni una respuesta HTTP. Devuelve solo el resultado codificado o lanza un error de Gateway que el adaptador RPC de Connection mapea para el transporte.

## La cadena de llamadas `/api` compartida

Connection es dueño de una ruta `/api` en el servidor HTTP. El Gateway monta una prueba síncrona de titularidad de endpoints y el handler RPC Remote en Connection:

```text
ctx.connection.rpc.intercept(
  '/api',
  endpoint => ownsRemoteEndpoint(endpoint),
  (endpoint, payload, signal) => {
    const { namespace, method } = parseEndpoint(endpoint)
    const { args } = parsePayload(payload)
    return ctx.typertGateway.invoke({ namespace, method, args, signal })
  },
)
```

El Gateway reclama un endpoint cuando el registro del Host contiene su descriptor estricto, recuerda un descriptor estricto retirado o encuentra un marcador `@Remote` que coincide en un binding SRC de Service activo. Un endpoint reclamado permanece en el Gateway después de que fallen la decodificación del payload, la resolución del descriptor o la invocación; solo un endpoint que no es propiedad de Remote llega al fallback del API Proxy heredado.

La mitad de Host de Connection pasa un FetchHandler compuesto al puente HTTP. Cuando el puente crea un `Request` estándar, ese handler selecciona el FetchHandler RPC del Gateway o el FetchHandler del API Proxy. Ambas rutas reutilizan el mismo envelope de solicitud/respuesta, rpcId, serialización, confianza, errores de transporte y `RpcError`. El mapeo físico actual es:

```text
POST /api/<namespace>/<method>
```

El payload Remote es un objeto JSON con nombre, no un array posicional, y no lleva un `InvocationDescriptor`. Una llamada Goal normal tiene esta ranura de payload:

```json
{
  "args": {
    "agentId": "session-1",
    "request": {
      "objective": "finish the migration"
    }
  }
}
```

El recorrido completo es:

```text
ctx.remote.goals.create(sessionId, request, signal?)
→ Client InvocationDescriptor 编码 { args: { agentId, request } }
→ Client 合并 caller signal 与 contribution mount lifetime
→ ctx.connection.rpc.call('/api', 'goals/create', { args }, signal)
→ Connection 创建 rpcId 和既有 client-request envelope
→ 当前 carrier 发送 POST /api/goals/create
→ Connection Host half 执行共享 trust，再由 bridge 创建标准 Request
→ 复合 FetchHandler 判断 endpoint ownership 并选择目标 FetchHandler
→ Typert interceptor 调用 ctx.typertGateway.invoke(..., request.signal)
→ Host InvocationDescriptor 解码、lookup、receiver 解析并把 signal 注入 Reflect.apply
→ result codec 编码
→ Connection 写入既有 RPC result 并回送相同 rpcId
→ Client result codec 验证并返回 CreateGoalResult
```

Remote no define una respuesta de segunda capa `{ ok, value/error }`. Los valores con éxito y los errores de Gateway usan directamente el `result` de la respuesta RPC existente. El adaptador convierte los fallos ordinarios de Gateway y de invocación de negocio al envelope `RpcError` existente con `code: 'internal'`; un error RPC existente que un resolver transporta en `TypertLookupFailure` se devuelve sin cambios, preservando códigos de error estables para los fallos de reanudación en frío y las vallas de propiedad. La categoría estructurada de errores del Gateway sigue disponible solo dentro del proceso, mientras que el mensaje transporta el diagnóstico a través de Connection.

El Gateway no gestiona permisos por método, identidad del llamador, idempotencia ni estado de conexión de larga duración. Solo propaga la cancelación cooperativa desde Connection hacia los métodos de negocio explícitamente conscientes de la cancelación. Los endpoints Typert usan la política de host de confianza de Connection; los endpoints no reclamados conservan las políticas de confianza y de métodos privilegiados del API Proxy heredado. La migración a WebSocket de Connection sigue siendo trabajo de seguimiento aparte.

## Connection y los límites del protocolo

El Service Remote de cliente es dueño de las contribuciones Remote, de la materialización de Services de espacio de nombres, del binding de Scope y de la correspondencia entre parámetros posicionales y descriptores. El Gateway es dueño de los descriptores del Host, de la titularidad de endpoints, del lookup, del Context y de la invocación de negocio. Connection envía `/api`, el endpoint y `{ args }` como una única llamada RPC al destino y devuelve el resultado RPC existente; no entiende Goal, Agent, lookup, descriptores ni los tipos Remote de cliente.

El Gateway registra en Connection solo su comprobador de titularidad y su handler RPC; no registra una ruta HTTP. Connection monta la ruta `/api` compartida en el servidor HTTP y le da al puente un FetchHandler compuesto; ese handler despacha los endpoints reclamados al Gateway y los no reclamados al API Proxy. Un transporte futuro de Connection puede preservar este orden sin cambiar el payload Remote, los decoradores de negocio, la DTS generada, los tipos de Remote API ni la interfaz de programación del Agent Scope.

## Límites de paquetes

- `@deepseek-ai/dsh-typert-protocol`: protocolos ligeros para decoradores, bindings, lookup, Remote Scope y descriptores.
- Generador de Typert: analiza los Programas del Host/Cliente, genera las caras locales y las proyecciones Remote de consumo, y emite la información canónica de símbolos/Zod.
- Runtime de Typert: almacena por separado la reflexión local del entorno actual y las contribuciones Remote importadas.
- `@deepseek-ai/dsh-api-gateway`: su entrada por defecto asocia las definiciones del Host con Services, reclama endpoints Remote, realiza el lookup, resuelve los receptores de Context, invoca métodos, codifica resultados y registra un interceptor `/api` en Connection; su entrada `/client` monta las contribuciones Remote, crea Services y métodos estrictos de espacio de nombres Remote y delega las llamadas en `ctx.connection.rpc`. Las entradas comparten el protocolo Remote pero no importan las fusiones de interfaces de Cordis del otro.
- `@deepseek-ai/dsh-api-remotes`: la capa BFF; es dueño del resolver de Agent/Session del Host, selecciona las contribuciones `/remote` del cliente y expone los tipos Remote fusionados a los paquetes de negocio a través del contrato compartido `TypertClientRemote`.
- Connection: es dueño del único carrier servidor HTTP/WebSocket futuro, de la ruta `/api` compartida y del FetchHandler compuesto, del fallback del API Proxy, del envelope RPC, del rpcId, de la serialización, de la confianza y del transporte de errores.
- Paquetes de objetos de negocio como Agent/Session: son dueños del lookup, de los providers de Context, de los tipos de ID canónicos y de las entradas públicas de solo tipos.
- Composición del API Proxy del Host: suministra los valores por defecto de Web Agent y la configuración de scope a API Remotes y consume el mismo `agentFor()` para los métodos heredados.
- Paquetes de Services de negocio: declaran bindings, métodos Remote y sus tipos de solicitud/resultado, y exportan el subpath `/remote` generado.

## Alcance publicado y trabajo diferido

La ruta vertical publicada es `@deepseek-ai/dsh-goal/remote → Browser Client Remote → Connection RPC /api → Host Gateway → GoalService.remoteExportCreate()`. El mismo descriptor directo con un lookup de Agent admite tanto `ctx.remote.goals.create(agentId, request)` como `agentCtx.remote.goals.create(request)`. Las sesiones frías ordinarias se reanudan a través de `agentFor()` durante el lookup, mientras que las identidades propiedad de subagentes conservan la valla `agent-busy` existente; `@RemoteScope('agent')` sigue siendo el modo distinto de receptor con ámbito.

Connection suministra el interceptor de canal compartido y el mapeo actual del carrier HTTP. La migración a WebSocket, el runtime y el carrier del TUI, el cableado del Agent Scope del TUI, las máquinas de estado de Permission/Approval, los flujos de eventos de Session, la autorización de llamadas, los reintentos, la idempotencia y la compatibilidad de protocolo entre versiones quedan fuera de esta decisión.

La topología de paquetes es `api/remotes → api/gateway → client/connection → host/webserver`. Connection y WebServer conservan sus rutas actuales en este cambio; moverlos más adelante a `api/connection` y `api/webserver` cambia la ubicación de los paquetes, no estos límites de Services. El API Proxy heredado permanece igualmente bajo `host/apiproxy` como fallback para los métodos aún no migrados a Remote.

## Alternativas consideradas

**Seguir usando el paquete central de API Proxy.** Esto exigiría declarar los métodos de negocio, las rutas del Host y las interfaces de cliente repetidamente en varios lugares. También mantendría las llamadas directas, las interacciones con estado y los flujos de eventos atados al mismo ciclo de vida, así que esta alternativa se rechaza.

**Realizar la reflexión estricta mediante decoradores en runtime.** Los decoradores de JavaScript no pueden recuperar los tipos de TypeScript borrados, la identidad de los símbolos públicos ni los codecs Zod completos. Inyectar un símbolo privado del compilador en un constructor también ocultaría las dependencias reales de la clase de negocio, así que Typert genera la información estricta en tiempo de compilación.

**Usar un preload, un hook de loader o un `ts.Program` completo durante el arranque SRC.** Esto podría reutilizar el análisis LIB, pero añadiría requisitos a cada entrada de arranque desde código fuente. SRC solo necesita un descriptor permisivo utilizable, así que usa marcadores de decoradores, nombres de parámetros de funciones y providers explícitos; las comprobaciones estrictas permanecen en la pasada del contrato LIB.

**Escribir a mano la interfaz del cliente.** Una interfaz escrita a mano no puede garantizar que contenga solo métodos marcados como Remote y puede desviarse de las firmas del Host, de los IDs de lookup y de los schemas Zod. Por eso los tipos de cliente se proyectan automáticamente desde el Programa del Host.

**Usar un plugin del language service/compilador de TypeScript para que el cliente entienda los decoradores directamente.** Esto exigiría que los editores, Vite, tsc, tsx y los consumidores publicados instalaran un plugin adicional, haciendo la integración demasiado invasiva. El diseño genera en su lugar archivos `.d.ts` ordinarios y mapas de declaraciones estándar.

**Importar la DTS completa del Host en el cliente o en el TUI.** Esto arrastraría los Services del Host y las fusiones de interfaces de Cordis mientras expone métodos sin marcar a los consumidores. La DTS Remote remite solo a símbolos públicos de solo tipos y aumenta mapas Remote dedicados.

**Generar solo la DTS Remote, sin JS.** Los tipos funcionarían, pero el runtime no podría enumerar endpoints, codecs y modos de Context sin un Proxy u otro registro escrito a mano. La misma proyección del Host emite por tanto también una contribución Remote JS.

**Dejar que un import de nivel superior `/remote` registre estado global implícitamente.** El Context de Cordis de destino puede no existir cuando se produce la evaluación ESM, y la propiedad se vuelve ambigua entre varios Contexts, HMR y la disposición. Un import de valor normal devuelve por tanto solo una contribución, que el assembly del entorno monta explícitamente a través del Service Remote de cliente.

**Crear un transporte, una ruta HTTP o un canal `/api2` separados para Remote.** Esto duplicaría o dividiría la propiedad del Server de Connection, el rpcId, la serialización, la confianza, los errores y el futuro ciclo de vida del WebSocket. El interceptor `/api` compartido mantiene en su lugar una única ruta física y deja que Connection conserve el API Proxy como FetchHandler de fallback.

## Verificación

- El Goal Service decora directamente los métodos de mutación cuyas firmas de negocio ya coinciden con el contrato Remote y conserva `remoteExportCreate(...)` solo para adaptar `GoalView` a `CreateGoalResult`, sin una segunda ruta, codec ni lista de métodos del cliente.
- Un `build:lib` limpio emite los artefactos Remote del Host y del consumidor antes de la compilación del cliente, incluidos el JS, la DTS y el mapa de declaraciones del paquete de negocio bajo `/remote`.
- Después de `clean`, los `typecheck`, `lint` y `doc-typecheck` independientes regeneran los contratos Remote; el hook pre-push usa el mismo typecheck preparado, y los consumidores de código fuente de CI esperan una única pasada de contrato compartida.
- Importar `@deepseek-ai/dsh-goal/remote` añade el tipo estricto `ctx.remote.goals.create(...)` y la navegación de declaraciones hasta `remoteExportCreate`; omitir ese import omite el espacio de nombres.
- Montar la contribución JS del mismo import suministra la reflexión de endpoint, parámetro, resultado, lookup, Context y Zod y materializa la llamada sin un stub escrito a mano.
- Las llamadas raíz y con ámbito de Agent cruzan el carrier `/api` compartido real, resuelven `agentId` al Agent vivo, invocan el receptor Goal original y regresan a través del envelope RPC existente.
- Los lookups de Agent y Session comparten una única reanudación de sesión fría en vuelo; las sesiones frías ordinarias reciben objetos restaurados, mientras que las identidades de subagente, tanto frías como vivas, devuelven `agent-busy` antes de la invocación de negocio.
- Los artefactos y mapas Remote contienen solo métodos marcados y ninguna dependencia del navegador, preservando el mismo límite de consumo para un TUI futuro.
- Las pruebas de ciclo de vida retiran y vuelven a montar descriptores, Services, lookups, providers de Context y espacios de nombres del cliente; las dependencias no disponibles fallan sin llamadas obsoletas ni fallback de ID en bruto.
- Las pruebas de cancelación cubren la generación estricta, el reconocimiento del nombre final en SRC, la fusión de señales del cliente, la propagación de Connection al Gateway y la inyección del Host fuera de los `args` wire.
- Los endpoints no reclamados continúan por la ruta existente del API Proxy con su confianza, sus métodos privilegiados, su Permission/Approval y su comportamiento de flujos de eventos de Session sin cambios.

## Consecuencias

Los tipos de Remote API dependen de las declaraciones `lib` generadas. La orquestación de builds y verificaciones debe completar la pasada del contrato del Host antes de compilar o analizar semánticamente a los consumidores del Host y del cliente; un orden incorrecto hace que un comando limpio dependa de artefactos obsoletos.

La navegación de código fuente exige que un paquete Remote publique tanto su mapa de declaraciones como el archivo `src` al que remite el mapa. Si el `files` del paquete omite cualquiera de los dos lados, los tipos siguen compilando pero la navegación del consumidor se detiene en la DTS generada. La comprobación del manifest del workspace debe tratar por tanto ambos como un único contrato de publicación.

El descriptor SRC permisivo no valida la estructura interna del JSON ordinario. Cuando cambia una firma Remote del Host, el Web y los consumidores de tipos estrictos deben reconstruir la lib porque no existe un watcher incremental del contrato.

Los tipos públicos canónicos exigen que los DTO de negocio tengan entradas de solo tipos, lo que puede exponer paquetes cuyos tipos del Host y entradas de implementación están actualmente mezclados. La build rechaza esos límites en lugar de copiar los tipos para ocultarlos.

Los imports de tipos y las contribuciones en runtime tienen efectos distintos. `import type {}` extiende solo la superficie Remote estática. Si un entorno de llamada real omite la contribución de valor, el Service Remote de cliente debe fallar con un error explícito de «Remote not mounted».

El navegador y el Host mantienen cada uno sus propias instancias de Zod y no pueden comparar identidades de objetos entre realms. La consistencia está garantizada solo por las claves de símbolo canónicas, el mismo modelo generado y el comportamiento wire.

Un consumidor puede importar un contrato Remote que no está montado actualmente en el Host. Los tipos significan «este protocolo de capacidad fue seleccionado por el consumidor», no que un Service correspondiente exista actualmente en el proceso de destino; un endpoint no disponible debe fallar explícitamente en runtime.

La API de canal general de Connection debe servir tanto al carrier HTTP actual como a un carrier WebSocket futuro. Si el Remote de cliente o el Gateway exponen `fetch`, una solicitud HTTP o un handle de ruta, la migración a WebSocket volverá a perforar la capa Remote. Esos objetos físicos deben por tanto permanecer internos a Connection.

Los endpoints Remote usan la autoridad `trusted-host` de Connection. El loopback se acepta por defecto y los llamadores de LAN requieren una configuración explícita de trusted-host, pero esta capa no añade autorización del llamador por método; todo host de confianza puede invocar un endpoint Remote montado.

`hasSeen()` favorece la seguridad de la definición estricta sobre la disponibilidad SRC. Mientras un descriptor estricto está retirado, como durante HMR, el Gateway sigue reclamando el endpoint y lo reporta como no disponible en lugar de recurrir a un descriptor SRC débil. Volver a registrarlo lo restaura; solo un reinicio del registro de Typert olvida la definición estricta histórica.

Las firmas Remote conscientes de la cancelación reciben el `AbortSignal` de solicitud de Connection, de modo que una desconexión HTTP o una cancelación del lado del cliente llega al trabajo de negocio en curso sin entrar en el protocolo JSON. La cancelación sigue siendo cooperativa: los métodos sin el parámetro final reservado continúan ejecutándose, y un método que recibe la señal debe pasarla a sus propias operaciones cancelables u observarla directamente.

La configuración de lookup opera actualmente con granularidad de clave, de modo que todo parámetro `agent` o `session` usa la misma política de reanudación en frío. Un Remote específico que requiera semántica solo-en-vivo debe esperar una política explícita por parámetro o por endpoint; no se puede dejar que la implementación de negocio adivine si el objeto acaba de ser reanudado.
