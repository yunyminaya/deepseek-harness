# Agent Note: Capas de la GUI y protocolo RPC: división host/client por proveedor de capacidades, modelo de mensajes de cuatro cuadrantes y carrier fetch

Status: implemented

[English](2026-07-19-gui-layering-and-rpc-protocol.md) | Español

> División de trabajo: este documento = el modelo de capas + el protocolo RPC independiente del canal; la implementación Web del protocolo combina la subida HTTP con [el carrier de bajada WebSocket](2026-08-04-websocket-downlink-carrier.es.md), mientras que la capa de objetos del navegador está en [la nota de arquitectura del cliente web](2026-07-19-gui-web-client-architecture.es.md).

## Problema

Necesitamos una capa de integración de UI. Más allá de la línea base ACP/stdio existente, vienen más clientes de producto — Web (server), Electron y otros. Los llamamos Client y queremos las siguientes capacidades:

- Un único proceso `dsh` que soporte a la vez `dsh web` (serve) y `dsh --profile headless` (headless) — un proceso, dos modos (una reserva de diseño)
- Arrancar dentro de Electron con las mismas tecnologías Web que `dsh web`

Eso exige un modelo estable de responsabilidades por capas en el codebase de ingeniería, para que los clientes futuros se conecten limpiamente.

Al mismo tiempo, los canales físicos difieren según el consumidor (HTTP/WebSocket del navegador, fetch/SSE en proceso, IPC más adelante), así que también necesitamos un modelo de mensajes independiente del canal y una única fuente de verdad del contrato — «añadir un método» y «cambiar de carrier» no deben enredarse entre sí, y cada mensaje del cable (wire) debe poder validarse por tipos, observarse y reconciliarse.

## Decisión

### Capas

Los directorios se dividen en capas así:

- `packages/host/*`: los paquetes aportan solo capacidad del lado host (representan el núcleo de ingeniería Node.js construido sobre el sistema de plugins del harness existente), y además
    - el protocolo unificado del backend (fetch, HTTP, interfaces de streaming…) — definiciones y soporte, ver las secciones «Message protocol» más abajo
- `packages/client/*`: los paquetes aportan solo capacidad del lado cliente; cada paquete se mantiene de un solo lado. Aquí viven tres clases (los ejes los posee [la nota de carga de plugins del cliente](2026-07-23-client-plugin-loading-model.es.md)):
    - **Librerías puras** (`ui-slots`, `ui-primitives`, más el paquete kernel `loader`): paquetes ordinarios de índice raíz, empaquetados estáticamente en el shell; las dos librerías de cliente se siembran en la tabla de módulos.
    - **Paquetes de entrada de llegada estática** (`connection`, `runtime`, `ui-theme`, `i18n`, `hmr`): sin clave `dsh.client` y sin bundle de navegador — el shell empaqueta su mitad `src/client/` y la registra con `ctx.modules`; se gobiernan como entradas del grafo escrito por el host como todo lo demás.
    - **Paquetes plugin de llegada por fetch** (`ui-layout`, `ui-sidebar`, `ui-conversation`, `ui-trajectory`): doble entrada — el índice raíz es la mitad node (un `apply` vacío, que existe para que el Loader del host gobierne el ciclo de vida y el registro web de plugins descubra la declaración `dsh.client` del package.json); la implementación vive bajo `src/client/` y se publica como la subruta `./client` (un bundle de fábrica de clausuras tsdown). El consumo de `/client` entre plugins es solo de tipos; la cooperación de valores pasa por los servicios de Cordis.
- `apps/` contiene las aplicaciones exportadas externamente, ensambladas a partir de mezclas de Client / Host.
    - `apps/web` (`dsh-web-frontend`) es la aplicación vite: un `main.ts` delgado sobre la API del shell exportada por `dsh-client-web`.
    - `apps/cli` (`@deepseek-ai/dsh`) despacha comandos: `dsh web` = Host + webserver + el dist de `dsh-web-frontend` compilado; `dsh --profile headless` = [un punto de entrada directo del núcleo Agent/Session](2026-08-09-headless-direct-core-entry-point.es.md), con cero capas de Host, HTTP o navegador.
    - Una futura aplicación Electron reutiliza los mismos paquetes de cliente web sobre un carrier de fetch por IPC.

```
apps/*  (applications: apps/web = vite app, apps/cli = bin dispatch)
  │ consume
  ▼
packages/host/*                      packages/client/*
  apiproxy   front layer: protocol     pure libs: ui-slots / ui-primitives
  runtime    assembly / host entity    dsh.client plugins ×8 (node half = empty apply,
  webserver  Web HTTP carriage                              client half = src/client/)
  │ ctx.plugin(...)                      ▲ import only apiproxy's /api /client subpaths
  ▼                                      │ (type-only + the client base class)
harness core packages ──────────────────┘ (types reach the browser via import type)
```

Disciplina de direcciones (cada regla es auditable desde las dependencias de paquetes):

- `runtime → apiproxy` es unidireccional; apiproxy depende solo de definiciones de tipos.
- Los paquetes del lado cliente **nunca importan** el runtime de los paquetes del lado host (consumen solo las dos subrutas seguras para navegador `/api` y `/client`).
- `webserver` no depende de `runtime`: aporta una implementación de la interfaz `{ fetch }` — «webserver ← runtime» es una relación de inyección en tiempo de ejecución, no una dependencia de paquete.
- Los imports de cliente entre paquetes usan la subruta `/client` para los paquetes plugin, y entre paquetes plugin son solo de tipos — un import de valor entre plugins es un error de build en el gate de pureza de tsdown (la cooperación de valores pasa por los servicios de Cordis; [la nota de carga de plugins del cliente](2026-07-23-client-plugin-loading-model.es.md) posee las reglas de aristas).

TypeScript comprueba en **dos programas agregados** referenciados por una raíz de solución (`tsconfig.json` = solución; `tsconfig.host.json` = lado host + pruebas, excluyendo `packages/client`; `tsconfig.client.json` = paquetes de cliente y sus pruebas): ambos lados fusionan la interfaz `Context` de Cordis bajo las mismas claves (`sessions`, `loader`) con servicios distintos, de modo que un único programa vería ambas fusiones de declaraciones y reportaría una colisión. Las hojas compartidas (session/llm/tools/apiproxy…) se compilan una vez y las referencian ambos programas ([topología](../process/2026-07-22-tsconfig-solution-root-two-aggregates.es.md)).

Del lado del protocolo: interfaces TS (`packages/host/apiproxy/src/api/`, cero dependencias de Node, importables desde navegador); los mensajes de wire se unifican bajo un modelo **bidireccional** — cada mensaje lógico se clasifica por «quién inicia × petición/respuesta» (dos ejes, cuatro celdas, llamados los cuatro cuadrantes más abajo), desacoplado del canal físico; todos los clientes heredan de `AbstractApiClient` (los invariantes del protocolo viven enteramente en la clase base; las diferencias de plataforma son solo el aspecto de transporte `doFetch`).

#### Roles de las capas

| Capa | Paquete | Responsabilidad | Disciplina clave |
|---|---|---|---|
| Capa frontal | `dsh-host-apiproxy` | Definiciones TS/zod (api/) + la abstracción de fetch (fetch/: handler + clase base del cliente) | Mantenerla simple — todo consumidor la necesita; importable desde Node y desde navegador por igual; el contenido del protocolo en las secciones «Message protocol» más abajo; los clientes no deben saltarse api a través de ctx |
| Capa de ensamblaje | `dsh-host-runtime` | Composición de plugins + integración de ApiProxy + el montaje de los plugins de UI web (árbol Loader en memoria sobre los ocho paquetes dsh.client); hogar de la configuración a nivel de host (defaults/persistenceRoot, perfil de usuario futuro) | Qué plugins se montan y con qué defaults se decide solo aquí; los shells no deben alterar el ensamblaje |
| Capa carrier | `dsh-host-webserver` | HTTP Web y upgrade: servido estático + reenvío de `/api/*`→handler + ruta de upgrade WebSocket + semántica de cierre; endpoint de bundles de plugins + inyección del manifest `__DSH_BOOT__` (alimentado por el registro web de plugins) | Solo Web (acceso desde navegador); cero dependencias de workspace (el registro llega por inyección estructural); Electron no lo reutiliza |
| Librerías de cliente | `dsh-client-ui-slots` / `dsh-client-ui-primitives` | Contratos de slots / átomos React puros | Sembradas en la tabla de módulos del loader por el shell |
| Plugins de cliente | `dsh-client-connection` / `dsh-client-runtime` / `dsh-client-ui-theme` / `dsh-client-ui-renderer` / paquetes de UI de funcionalidad | Árbol de plugins Cordis del lado navegador: consumidor de wire, servicios núcleo, theme, renderizado React y composición de funcionalidad — ver la nota de arquitectura del cliente web | Doble entrada (mitad node = apply vacío; implementación en `src/client/`); la cooperación de valores entre plugins usa servicios y slots |
| Aplicación | `@deepseek-ai/dsh` (apps/cli) + `dsh-web-frontend` (apps/web, la aplicación vite) | Despacho de binario grueso + un módulo de ensamblaje por aplicación (web.ts / headless.ts); la app vite es un main delgado sobre la superficie del shell `dsh-client-web` | Las aplicaciones usan imports dinámicos para no cargarse nunca entre sí; el conocimiento del workspace, como la ubicación de dist, se queda en la app |

#### Regla de nombres

Los paquetes bajo `packages/host/*` y `packages/client/*` **deben llevar el prefijo del grupo de directorio en el nombre del paquete**: host/runtime → `dsh-host-runtime`, client/runtime → `dsh-client-runtime`. El nombre del directorio no repite el prefijo del grupo (host/ ya lo expresa). La cola del nombre de paquete por tanto ≠ el nombre del directorio, de modo que el comodín `dsh-*` de tsconfig.base.json (que resuelve por nombre de directorio) no los alcanza — **cada paquete de estos dos grupos necesita una entrada paths explícita**, incluidas entradas separadas para las subrutas `/client` de los paquetes de cliente, para que la resolución a nivel de source coincida con el mapa de exports.

#### Cómo integrar una aplicación nueva (lista operativa)

1. **Elige una suplantación de fetch**: HTTP del mismo origen del navegador / inyección en proceso de `host.handler.fetch` / tu propia subclase de aspecto de transporte (p. ej. IPC de Electron futuro, ver la «tabla de subclases» más abajo).
2. **Escribe un módulo de ensamblaje bajo `apps/`**: `startHost()` + una subclase de cliente + la semántica privada de signal/print/exit de la aplicación; una mezcla nunca se convierte en un paquete — el ensamblaje se escribe en la app.
3. **Importa `dsh-host-webserver` solo si necesitas el transporte HTTP**; si no, cero puertos.

Las dos aplicaciones existentes preservan la división: la aplicación Web monta Host, carrier y composición del navegador, mientras que `dsh --profile headless` monta un runner de núcleo directo con cero Host, HTTP o puertos. Los puentes de protocolo clase ACP no siguen la lista de cliente-carrier: exponen el núcleo al ecosistema externo y se montan directamente con `ctx.plugin(entry-point plugin)` sin fetch.

## Protocolo de mensajes

Las secciones de aquí en adelante son el cuerpo del protocolo que transporta la capa frontal (`dsh-host-apiproxy`). El cable tiene exactamente cuatro clases de mensaje (los cuatro cuadrantes) — el transporte Web de la columna derecha es solo un ejemplo; cambiar de carrier (en proceso/IPC) deja los cuadrantes sin cambios:

```
                 client 发起                      server 发起
  request   ① ClientRequest                 ③ ServerRequest
            （POST /api/<method> body）      （WebSocket message：session 事件、审批/问答 requested）
  response  ② ServerResponse                ④ ClientResponse
            （该 POST 的 HTTP 应答体）        （POST /api/respond body，回填 ③ 的 rpcId）
```

### Formas completas del cable: una unión discriminada con nombre de cuatro miembros (`api/rpc.ts`)

| Tipo | Etiqueta discriminante | Campos | Propiedad de rpcId | Transporte Web |
|---|---|---|---|---|
| `ClientRequest` | `'client-request'` | `rpcId` `method` `payload` | lo acuña el cliente | cuerpo de `POST /api/<method>` |
| `ServerResponse` | `'server-response'` | `rpcId` `result` | hace eco de ① | el cuerpo de respuesta de ese POST (siempre HTTP 200) |
| `ServerRequest` | `'server-request'` | `rpcId` `method` `payload` | lo acuña el servidor | mensaje de texto WebSocket |
| `ClientResponse` | `'client-response'` | `rpcId` `result` | hace eco de ③ | cuerpo de `POST /api/respond` |

`RpcMessage = ClientRequest | ServerResponse | ServerRequest | ClientResponse`, estrechado con `switch (message.type)`.

**Disciplina de rpcId** (`RpcId` es una cadena con marca (branded string) con constructor `RpcId()`):

- Quien inicia acuña; una respuesta siempre hace eco del rpcId de la petición correspondiente y **nunca acuña un id nuevo**.
- Las server-requests se dividen en dos clases, distinguidas estáticamente por `method` (= el tipo de trama), **sin una tercera clase**: las tramas respondibles (`approval/requested`, `question/requested`) llevan un id de petición lógica estable (acuñado una vez al aceptarse, reutilizado tal cual en la repetición de línea base, con eco en la respuesta del cliente); las tramas de puro push (`session/event` etc.) llevan un rpcId que identifica ese único push (acuñado de nuevo cada vez).
- El código de negocio nunca acuña: la acuñación unaria se canaliza a la clase base del cliente `callUnary`, y la acuñación de tramas se canaliza al lado host.

### Formas estrechas de firma y completado del carrier

Las firmas de las interfaces de dominio solo perciben las formas estrechas: `RpcRequest<P> = { rpcId, payload }`, `RpcResponse<T> = { rpcId, result: RpcResult<T> }`. La capa carrier completa las formas estrechas hasta las formas completas (añadiendo la etiqueta `type` y `method`); la dirección nunca se infiere del canal. `RpcResult<T> = { ok: true; value } | { ok: false; error: RpcError }` — los métodos no lanzan errores de negocio.

### RpcReceipt: el recibo del carrier

El cuerpo de respuesta HTTP de una `ClientResponse` es `RpcReceipt = { accepted: true } | { accepted: false; reason: 'not-pending' | 'bad-response' }` — un recibo de la capa carrier, **no** un RpcMessage (una respuesta no tiene respuesta); las respuestas tardías o duplicadas reciben `not-pending`, y el punto de convergencia lógico son las tramas `*/resolved`.

## El sistema de tipos: las firmas son la fuente de verdad

### RpcMethodMap y genéricos derivados (`api/rpc-map.ts`)

Las estructuras de parámetros/retorno de los métodos **viven solo en las firmas de los métodos de la interfaz**; el mapa registra los propios métodos; cualquier otra posición (handler, cliente, store, pruebas) referencia los genéricos derivados — copiar literales o introducir tipos planos con nombre está prohibido:

```ts ignore-check
export interface RpcMethodMap {
  'session.list': SessionsApi['list']        // map key 即 wire 路径段
  // …其余方法同形登记，全集见 api/rpc-map.ts
}
// 派生泛型（穿透窄形取业务类型；实际声明带 K extends keyof RpcMethodMap 约束）
export type RequestPayload<K> = Parameters<RpcMethodMap[K]>[0]['payload']
export type ResponseValue<K> =
  Awaited<ReturnType<RpcMethodMap[K]>> extends RpcResponse<infer T> ? T : never
```

Los métodos de stream (`events.mux`/`events.host`) quedan fuera del mapa (no son unarios); `respond` también queda fuera del mapa (es una client-response, no una llamada de método).

### El modelo de errores (`RpcErrorDetailsMap`)

Una fila de ejemplo de un código de error:

| code | details | when |
|---|---|---|
| `bad-request` | `{ issues: ZodIssue[] }` | falló la validación zod del wire/payload |

El conjunto completo de códigos es `RpcErrorDetailsMap` en `api/rpc.ts`. `RpcError` es la unión distributiva expandida a partir del mapa: `code` discrimina y `details` se estrecha automáticamente tras un `switch`; **details es obligatorio** — un código nuevo = una fila del mapa + una rama del error-schema, y omitirlo es un error de compilación. Los fallos de transporte (red caída, host no levantado) los lanza el carrier como excepciones; las dos capas nunca se mezclan.

### Validación zod bidireccional y anclaje

- **Parseo de dos niveles**: el schema de forma completa una vez (estructura type/rpcId/method + el handler que comprueba path==method) → el payload de negocio se despacha por método/tipo de trama para un segundo parseo; el rechazo = `bad-request`.
- **Anclaje**: los schemas usan uniformemente `satisfies z.ZodType<Wire<T>>` (`api/rpc.schema.ts`). `Wire<T>` es un ensanchamiento profundo «| undefined» — el repositorio activa `exactOptionalPropertyTypes` mientras que zod `.optional()` emite `T | undefined`, de modo que anclar el tipo original es inviable en general; en el cable JSON, la ausencia y `undefined` son indistinguibles, así que el ensanchamiento no pierde semántica de validación. Las ramas anchas de paso directo (`SessionEvent`/`ContentBlock`/uniones de tramas/`RpcError`) y los schemas de ids con marca usan casts explícitos con comentarios.
- Los casts de marca tienen un único punto cada uno: cada archivo de schema canaliza su cast de id a un solo lugar (`rpcIdSchema` es el único punto de cast en rpc.schema.ts).
## La cara de contrato (ApiProxy)

La interfaz raíz es `ApiProxy = { sessions, host, events, respond }` (`api/index.ts`). Un dominio nuevo de client-request = un par de archivos nuevo (`<domain>.ts` + `<domain>.schema.ts`) + un campo de interfaz raíz + una fila del mapa.

### La tabla de métodos unarios

Una fila de ejemplo (la estructura de la tabla es la clave de lectura):

| method key | payload de petición | valor de retorno | semántica |
|---|---|---|---|
| `session.list` | `{ cursor?: string }` (cursor es un asiento reservado, sin implementar) | `{ items: SessionSummary[] }` | sesiones persistidas, updatedAt descendente; v1 no construye índice |

Los métodos restantes (`session.create`/`session.history`/`session.rename`/`session.prompt`/`session.cancel`/`host.describe`) no se recopiarán aquí — las firmas son la fuente de verdad; consulta `api/sessions.ts`, `api/host.ts` y `RpcMethodMap`.

### Tramas (server→client, uniones con nombre)

Dos streams lógicos: el stream mux (`/api/events.mux`, agregado de todas las sesiones) y el stream host (`/api/events.host`, eventos a nivel de host). El navegador consume un WebSocket de bajada por stream, mientras que el carrier fetch en proceso conserva SSE con el mismo enmarcado de eventos; el límite físico está en [el carrier de bajada WebSocket](2026-08-04-websocket-downlink-carrier.es.md). Una fila de ejemplo de trama:

| tipo de trama | payload | when |
|---|---|---|
| `session/event` | `{ sessionId; event: SessionEvent }` | paso directo del núcleo: los eventos del núcleo pasan tal cual, `assistant/chunk` ES el stream de tokens, no hay trama delta aparte |

Los tipos de trama restantes no se recopilarán aquí; las uniones completas son `MuxFrame`/`HostFrame` en `api/events.ts`. Tres puntos semánticos que conviene conocer: `session/subscribed` lleva lastSeq para detectar carreras de historial; las tramas requested de `approval/question` son respondibles (rpcId estable) y las tramas resolved son la superficie de convergencia; `host/agent-error` es la única salida para fallos en vivo sin posición de turno.

**Disciplina de paso directo**: los eventos/mensajes/bloques de contenido en el cable SON los tipos del núcleo (`SessionEvent`/`ContentBlock`) — no hay un segundo conjunto de DTO; los tipos llegan al navegador a través de la cadena de dependencias `import type`. `SessionEventMap` es extensible por fusión: el cliente aplica su default documentado (ignorar) a los tipos desconocidos, y el schema de eventos mantiene una rama «envelope válido + tipo desconocido» — el envelope se mantiene estricto; esto no es paso directo a nivel de campo.

### Semántica de sesión (compromisos del lado de la implementación)

- **Historial = repetición de eventos**: un solo fold (del lado del cliente); la paginación del historial y los incrementos en vivo comparten una única ruta de código; el servidor no mantiene un segundo sistema de instantáneas materializadas. Los límites de página del historial **se alinean con los límites de mensaje** (nunca cortan un mensaje por la mitad; los chunks se agrupan con su mensaje finalizado), y la última página incluye los chunks del parcial en vuelo.
- **Correlación del prompt**: el rpcId del prompt viaja en MessageSource (`'user-rpc'`) hasta el evento `user/message`; el cliente lo usa para promocionar el eco optimista.
- **Reconexión = reconstrucción**: no hay cursor de reanudación (la firma `since` de `mux` es un asiento reservado, ignorado si se pasa); al desconectarse, se reabre el stream y se vuelve a traer el historial; se compara `subscribed.lastSeq` con el seq de la cola del historial y se rellena una vez si hay un hueco.
- **El manejo de sesiones frías sigue la propiedad**: `session.history` y la lectura de origen para `session.fork` inspeccionan la persistencia sin un Agent, mientras que los métodos de sesión ordinaria ligados al Agent, como `prompt`, se reanudan a través de una tabla en vuelo deduplicada. Los subagentes respaldados por sesión rechazan esa ruta genérica de reanudación, y el estado de adjunto no se expone a los clientes (`running` ya lo cubre).
- **Aprobaciones/preguntas**: la trama requested acuña un rpcId estable al aceptarse; gana la primera respuesta, y la tabla pendiente en memoria del host (clave por rpcId) es el único árbitro; tras una reapertura de mux, las tramas requested aún pendientes se repiten después de la trama subscribed (rpcId reutilizado tal cual — recuperación por recarga). Los eventos de auditoría `approval/asked`/`decided` continúan por el log durable — las tramas = el plano de control en vivo, los eventos = la auditoría durable. **Estado**: el contrato y los tipos de trama están publicados; la tabla pendiente del lado host/el respondedor de wire no está implementada (`respond` en `api-proxy.ts` es un stub, siempre `not-pending`); PendingCard v1 es solo de visualización.
- **Sin versión de protocolo**: el cliente y el host se publican atados entre sí; `host.describe` no tiene campo protocolVersion; se introduce uno cuando aparezca un cliente con publicación independiente.
- **Disciplina de métodos reservados**: el mapa solo contiene métodos implementados; un método desconocido falla de forma ruidosa en el parseo del envelope (`bad-request`) — no hay código de respaldo not-implemented. La lista de reservas (implementar = copiar la firma en la interfaz de dominio + añadir la fila del mapa + añadir el par de schemas): `session.fork`, `prompt.mode` ganando `'inject'`, `task.list`, `host.listModels`, describe ganando `hostInstanceId`. (`session.rename` se graduó de esta lista: añade un evento `session/title` de origen de usuario.)

## El carrier del cliente: la familia de clases AbstractApiClient (`fetch/client.ts`)

**Los invariantes del protocolo viven en la clase base; las diferencias de plataforma son dos aspectos**: el método abstracto `doFetch(url, init)` (transporte) + el `onEnvelope` sobrescribible (observación).

### IApiClient: la vista del llamador

El mismo árbol de dominios que `ApiProxy`, pero los métodos unarios **reciben el payload de negocio directamente** — el carrier acuña el rpcId y envuelve el envelope; el código de negocio nunca acuña, y el código que necesite el rpcId de esta llamada lo lee del eco `RpcResponse` devuelto. `ApiProxy` es el contrato de firma en forma estrecha que implementa el lado de la implementación; `IApiClient` es la vista directa de payload que consumen los clientes; `AbstractApiClient` tiende el puente entre ambos. Los métodos se derivan por clave de `RpcMethodMap` — añadir una fila al mapa los actualiza mecánicamente.

### Rutas de protocolo que retiene la clase base

| Ruta | Contenido |
|---|---|
| `callUnary` | acuñar → tap → POST de la forma completa → parseo de `serverResponseSchema` → **comprobación de eco de rpcId** (el desajuste lanza) → tap → emitir la forma estrecha |
| `readSse` | fetch de streaming (no EventSource), enmarcado `\n\n`, concatenación `data:`, parseo de la forma completa de ServerRequest, tap, emitir la `RpcRequest<frame>` estrecha |
| `respond` | paso directo de client-response (el rpcId es un eco — nunca se acuña aquí); el cuerpo de la respuesta se parsea con `rpcReceiptSchema` |
| deadline unario | Las llamadas unarias ordinarias usan `AbortSignal.timeout` (default 30s, ajustable por constructor); `host.pickDirectory` y `command.execute`, con ritmo de usuario, omiten ese deadline pero conservan la cancelación del llamador/conexión; los streams no tienen deadline |
| `resolveBase` | navegador = el origen same-origin; entorno sin location (Node) = la autoridad ficticia `http://dsh.internal` |

### El aspecto de observación de envelopes a nivel de instancia

Las cuatro formas completas de los cuadrantes pasan por `onEnvelope`; la implementación base es un **buffer propiedad de la instancia con batching por microtareas** (las tormentas de tramas no deben molestar a los consumidores trama por trama; el estado a nivel de módulo se filtraría entre instancias/pruebas, de ahí que sea propiedad de la instancia). Los observadores se suscriben con `subscribeEnvelopes(listener)` (reciben lotes completos como `readonly RpcMessage[]` y devuelven una función de desuscripción); el lanzamiento de un listener queda aislado (la observación nunca debe morder al carrier). Sin suscriptores, el buffer no cuesta nada. Hoy ningún consumidor publicado se suscribe — el aspecto es el asiento designado para el diagnóstico del wire (el panel de depuración RPC retirado fue su primer consumidor, y uno futuro se conecta sin tocar el carrier).

### La tabla de subclases (transporte del canal)

| Subclase | Paquete | doFetch | Propósito |
|---|---|---|---|
| `InProcessApiClient` | apiproxy en sí | el handler `{ fetch }` inyectado | **El punto isomórfico**: `new InProcessApiClient(toFetchHandler(api))` nunca toca la red y, sin embargo, ejecuta la serialización real de wire, zod y el enmarcado SSE; las pruebas del carrier y los llamadores pueden ejercitar el protocolo sin abrir ningún puerto, mientras que el producto `dsh --profile headless` maneja el núcleo directamente |
| `WebApiClient` | dsh-client-connection | subida `globalThis.fetch` + un WebSocket de bajada same-origin por stream lógico | el cliente del navegador; límite físico en [el carrier de bajada WebSocket](2026-08-04-websocket-downlink-carrier.es.md) |
| `FixtureApiClient` | dsh-client-connection | no usado (override a nivel de protocolo) | desarrollo de UI sin servidor (`?fixture`): sobrescribe los virtuales `callUnary`/`openMux`/`openHost`/`respond` y es en sí mismo el servidor falso (los rpcIds de tramas los acuña él, con semántica autoconsistente) |
| subclase de puente IPC (ejemplo hipotético — no existe tal shell) | un shell Electron | ida y vuelta de serialización IPC | solo intercambiaría doFetch; el contrato y la clase base no cambian |

## Cómo extender (listas operativas)

**Añadir un método unario (5 pasos)**: ① añade la firma del método a la interfaz de dominio (parámetros/retorno en línea — es la única fuente de verdad); ② añade una fila a `RpcMethodMap`; ③ añade el par de schemas request/value en `<domain>.schema.ts` (anclado `Wire<RequestPayload<'…'>>`); ④ añade una fila al handler `UNARY_ROUTES` (el transporte Web del handler está en la nota de arquitectura del cliente web); ⑤ implementa en la implementación (haz eco de `request.rpcId`). Del lado del cliente, añade la fila de paso directo a las tablas de métodos de dominio de `IApiClient`/`AbstractApiClient`.

**Añadir un tipo de trama (3 pasos)**: ① añade una rama a la unión `MuxFrame`/`HostFrame` (las tramas respondibles deben anotar la semántica de rpcId estable); ② añade una rama al frame-schema; ③ el default documentado de fold/routing de los consumidores ya cubre los tipos desconocidos — añade una rama explícita cuando haga falta.

**Añadir un código de error (2 pasos)**: ① añade una fila a `RpcErrorDetailsMap` (details obligatorio); ② añade una rama discriminatedUnion a `rpcErrorSchema`.

**Conectar un carrier nuevo**: subclasea `AbstractApiClient` implementando solo `doFetch`; para interceptar a nivel de protocolo (como el fixture), sobrescribe en su lugar los virtuales `callUnary`/`openMux`/`openHost`. El contrato y la clase base permanecen sin cambios.

**Promocionar un método reservado**: copia la firma reservada en la interfaz de dominio → añade la fila del mapa → añade el par de schemas → añade la fila de UNARY_ROUTES → implementa.

## Consecuencias

Todo cliente consume un único contrato: añadir un método unario es un cambio mecánico de cinco pasos a partir de una sola firma, cambiar de carrier solo toca una subclase de `doFetch`, y cada mensaje del wire está validado con zod, es observable a través del tap de envelopes y es reconciliable por rpcId. Las llamadas unarias ordinarias siguen acotadas, mientras que `host.pickDirectory` y `command.execute` pueden permanecer pendientes hasta que la operación termine o llegue la cancelación del llamador/conexión; esto acepta que una operación no cooperativa con ritmo de usuario pueda colgar su petición en lugar de tratar la duración válida de una operación como un fallo de transporte. Los otros costes aceptados: dos grupos de paquetes necesitan entradas paths explícitas en tsconfig, y los métodos reservados (fork/inject/task.list/listModels/hostInstanceId) permanecen inactivos hasta que llegue un consumidor real.

## Alternativas consideradas

| Rechazada | Razón en una línea |
|---|---|
| Empaquetar por producto (una familia web, una familia electron) | Los productos comparten capacidades de host/cliente, no una implementación de aplicación; la división por proveedor de capacidades hace que una aplicación nueva necesite cero paquetes nuevos |
| Un paquete por mezcla (p. ej. un paquete headless independiente) | Una mezcla tiene exactamente un consumidor (su propia app); empaquetarla es abstracción sin dueño, mientras que el ensamblaje en la app es legible y desechable |
| Clientes consumidores que se conectan a ctx directamente (saltándose la capa apiproxy) | Los clientes exigen validación de wire, observabilidad y consistencia entre varios clientes. El headless directo es un punto de entrada local sin límite de cliente y usa los seams públicos de Agent/Session en lugar de un plano de comandos de cliente |
| webserver dependiendo de runtime (ahorrándose la inyección del handler) | La inyección por tipado estructural mantiene a webserver reutilizable por sidecars/pruebas con cero dependencias de workspace; una dependencia de paquete arrastraría conocimiento del ensamblaje a la capa carrier |
| Nombres de paquete sin el prefijo de grupo (continuando dsh-<tail>) | `dsh-runtime`/`dsh-web-ui` pierden su pertenencia en el espacio de nombres plano de npm; el coste es una entrada paths explícita por paquete |
| Reutilizar el JSON-RPC 2.0 del repositorio (dsh-sdk-jsonrpc-server) | Los códigos de error numéricos degeneran en un único código de respaldo, los contratos se alinean a mano en dos copias y los nombres derivan sin convención |
| Un modelo de tres envelopes (envelopes Request/Response/Frame, firmas ciegas a la dirección) | La correlación de rpcId es de la capa lógica; la semántica de dirección de tramas y respuestas inferida del canal se rompe en cuanto cambia el carrier |
| Pares de tipos Request/Response con nombre como fuente de verdad (mapa que registra pares de tipos) | Los tipos planos con nombre son un segundo nombre para el mismo hecho; la inferencia de firmas convierte añadir un método en un cambio en un solo lugar |
| Rutas estilo REST | El consumidor es nuestro propio cliente, sin expectativas REST de terceros; el mapeo RPC directo sobre la tabla de métodos es más mecánico |
| Una capa DTO (un segundo conjunto de estructuras solo para el wire) | Los tipos del núcleo llegan al navegador solo como tipos a coste cero; un DTO es un impuesto permanente de sincronización bidireccional |
| Reanudación por cursor (implementar mux since) | Reconexión = reconstrucción (estilo opencode) cubre todas las necesidades de v1; la firma conserva el asiento y la implementación espera a un consumidor real |
| Una función de fábrica createApiClient (la implementación original) | Las diferencias de plataforma (transporte/observación) son aspectos de herencia, no parámetros; la familia de clases permite que el fixture se sustituya a nivel de protocolo en lugar de envolver un envelope falso |
| Aplicar el deadline de transporte de 30 segundos a `command.execute` | La duración del comando es trabajo de la operación, no un presupuesto de salud del transporte; el deadline mata handlers válidos de larga duración, mientras que la cancelación del llamador/conexión ya aporta la ruta de detención requerida |
