# Agent Note: Arquitectura del cliente web: el árbol de plugins cordis del cliente, el sistema de slots y la capa de objetos sin React

Status: implemented

[English](2026-07-19-gui-web-client-architecture.md) | Español

> División de trabajo: el modelo de capas independiente del canal y el protocolo RPC (modelo de mensajes / sistema de tipos / cara de contrato / clase base del cliente) están en [la nota de capas y protocolo RPC](2026-07-19-gui-layering-and-rpc-protocol.es.md); este documento = el lado del navegador: cómo se carga el árbol cordis del cliente, cómo se componen los plugins de UI mediante slots y servicios, y cómo la capa de objetos sin React alimenta a React a través de instantáneas inmutables.

## Problema

Dos fuerzas dan forma al cliente de navegador. Primera, el streaming: en una UI de conversación impulsada por eventos, si el estado de negocio (la ventana de eventos, la acumulación de streaming, las interacciones pendientes, la máquina de estados de la conexión) se dispersa por los componentes de React y un store global, cada trozo de tokens sacude el árbol de render, y cambiar de librería de UI significa reescribir la lógica de negocio. Segunda, la modularidad: las funcionalidades de UI (layout, sidebar, conversation, theme, locale) deben ser plugins cargables de forma independiente — compuestos en tiempo de ejecución a partir de un manifest servido por el host, no compilados en un único bundle — sin renunciar a la seguridad de tipos en tiempo de compilación a través de los límites de los plugins.

## Decisión

Ambos extremos ejecutan cordis. El host es un árbol de plugins cordis; el navegador ejecuta un segundo árbol cordis del lado del cliente cuyo cada capacidad de UI es un plugin cargado dinámicamente por un loader retenido por el shell. Dentro de ese árbol, el ctx de Cordis aloja todos los hechos de tiempo de ejecución (servicios, stores, ámbitos de sesión) y React es proyección pura: los componentes no importan nada del framework, lo reciben todo a través de props y se suscriben a instantáneas inmutables con `useSyncExternalStore` (uSES en adelante).

```
┌─ Host ─────────────────────────┐   ┌─ Browser ─────────────────────────────────────────┐
│ sessions/agents/SessionLog     │   │ client cordis root ctx                             │
│ apiproxy: RPC + mux/host 双流  │◀─▶│  ├ vendored Loader + ctx.modules（内核，壳静态持有）│
│ webserver:                     │   │  ├ immediately entries: connection/runtime/        │
│  ├ GET /plugins/<id>/client.js │   │  │   ui-theme/i18n（fetch bundle，boot 预拉）       │
│  └ GET / 注入 __DSH_BOOT__ 图  │   │  ├ lazy entries: layout/sidebar/                   │
│                                │   │  │   conversation/trajectory（fetch bundle，按需） │
└────────────────────────────────┘   │  ├ ui-renderer（fetch bundle，React 根）       │
                                     │  └ session scope ×N（观看驱动，惰性建）            │
                                     │ DOM loading 页 → settled → React UI 一次成型       │
                                     └────────────────────────────────────────────────────┘
```

## El árbol cordis del cliente y la cadena de carga

La cadena de carga — las dos clases de paquetes (normal vs plugin dsh.client), la división módulo-sistema/gobernador de plugins, el arranque de dos fases sobre el grafo de entradas escrito por el host con revisiones, y la recarga en caliente — es propiedad de [la nota de carga de plugins del cliente](2026-07-23-client-plugin-loading-model.es.md). Los hechos que sostienen la carga de este documento: el navegador arranca el mismo `@cordisjs/plugin-loader` vendido que el host, con un sistema de módulos de cliente (`ctx.modules`, `packages/client/modules`) que llena su contrato `internal`; cada unidad con comportamiento de producto es una entrada del grafo `__DSH_BOOT__` escrito por el host — cada paquete de plugin de producción (infraestructura incluida) lleva la declaración `dsh.client` y llega como un bundle de clausuras tsdown `./client` traído por fetch, y las filas `immediately` solo difieren en el prefetch de la fase uno del arranque, mientras que los paquetes normales (la familia react, cordis, las librerías aún no promocionadas) siguen empaquetados en el shell, sembrados e invisibles para el grafo; los bundles ejecutan `window.__ModuleLoader__.load({ id, factory })` y su `require` se responde desde la tabla de módulos CJS perezosa (palabras semilla + fábricas registradas, materializadas y memoizadas en el primer require — los imports de valores entre plugins son un error de compilación; la cooperación pasa por los servicios de Cordis); los estilos globales y los CSS Modules se incrustan en el bundle del plugin propietario y se inyectan como `<style data-plugin="<id>">` al materializarse (los CSS Modules también reciben nombres con hash; las etiquetas de propiedad hacen posible la eliminación en recarga); la recarga en caliente está viva en los grafos de desarrollo — el webserver hace stat-poll de los bundles que sirve y emite tramas SSE `rebuilt`, y el plugin `client-hmr` intercambia una fibra por trama. Tras `loader.await()` y un barrido con todo en ACTIVE, el kernel sin framework llama una vez a `ctx.uiRenderer.mount(container)` del renderizador de UI dinámico — se crea cada entrada y cada fibra llega a ACTIVE, con las fibras FAILED/PENDING listadas de forma ruidosa; no existe un modo de disponibilidad parcial (el renderizado progresivo es trabajo diferido).

Los universos de tipos se mantienen separados a nivel de agregados — `tsconfig.host.json` es el programa host y `tsconfig.client.json` el programa cliente, ambos referenciados por la raíz de solución `tsconfig.json` — porque ambos lados fusionan el `Context` de Cordis bajo las mismas claves (`sessions`, `loader`) con servicios distintos; los paquetes de cliente consumen el vocabulario del wire a través de subrutas solo de tipos (`@deepseek-ai/dsh-session/types` y similares), de modo que ninguna aumentación del host viaja al programa cliente.

## El sistema de slots: cómo se compone la página

El sistema de slots tiene su propia nota — [el estándar del sistema de slots](2026-07-22-slot-type-chain-implementation.es.md) — y este documento se remite a ella por completo. El resumen de un párrafo para orientarse: ui-renderer solo renderiza `'root'`; un plugin compone la UI mediante una única llamada `register` que ocupa un slot, declara y autoriza sus slots hijos (objeto spec `children`), declara su store e inyecta su cara de negocio; las props de los componentes llegan en cuatro tramos derivados automáticamente (`PropsRuntime<K>` / `PropsRenderSlots<S>` / `PropsStore<H>` / inject), cada uno desde su única fuente de verdad. La fusión de declaraciones de `SlotMap` es la autoridad de tipos y las entradas solo llevan el tramo del propietario («quien lo inyecta es dueño de su tipo»); cada entrada renderizada vive en un error boundary por entrada.

Dónde vive la implementación: el núcleo del registro y los tipos de tramos de props viven en `packages/client/ui-slots`; el renderizador de salida, el puente uSES, la instalación a nivel de aplicación y el montaje de la raíz viven en `packages/client/ui-renderer`.

## Servicios y direccionamiento de ámbitos (scope)

Un servicio es la única API de un plugin hacia otros plugins (los componentes de UI y las caras de inyección no son APIs; un plugin al que nadie llama no monta ningún servicio — ui-trajectory es el ejemplo de plugin mínimo: ningún servicio de ctx, solo registros de view-slots). El catálogo: `ctx.connection` (cliente api + manejos de streams), `ctx.slots` (envoltorio del registro que emite `slots/changed`, entrada de render, contrato de instalación del renderizador), `ctx.sessions` (store de la lista, estado de la sesión actual, árbol de ámbitos), `ctx.loader`, `ctx.theme`, `ctx.i18n`, `ctx.layout` (navegación de vistas entre plugins), `ctx.conversation` (send/cancel/startSession). El estado de visualización que antes vivía en los stores de servicios (anchos de panel, selección, borradores) ahora vive en stores declarados por las entradas, según [el estándar del sistema de slots](2026-07-22-slot-type-chain-implementation.es.md).

No hay modelo de registro de componentes aparte de los slots — los antiguos anillos de vista y de herramientas se disolvieron ambos en él. Las vistas de conversación son entradas del slot de lista `'conversation.view'` que declara ui-conversation, los metadatos de pestaña viajan en las opciones de registro (`id`/`order`/`label`), y el chrome por vista vive dentro de los propios componentes de vista. Los Nodes de negocio del Chat final se despachan por el slot con clave/sesión `'conversation.chat.node'`; ui-tool es dueño de su entrada `tool-call`, renderiza recursivamente los `subCalls` suministrados y declara el slot hijo con clave/sesión `'tool.call.toolview'`. El espacio de claves permanece abierto en tiempo de ejecución (SlotMap declara slots, nunca claves), y las raíces y descendientes despachan por `entryKey: toolName` con `GenericToolCard` como respaldo. Los paquetes de negocio registran vistas atómicas con `ctx.slots.inject('tool.call.toolview', () => ctx.slots.register({ name: 'tool.call.toolview', key: '<tool>' }, Row))`; la declaración es la dependencia de carga y recarga ([decisión](2026-08-05-slot-declaration-injection.es.md)). ui-conversation delega aparte el cuerpo de detalles de la llamada seleccionada a través de `'conversation.details.tool'`, de modo que los modelos de tarjeta de ui-tool siguen siendo el único propietario de la presentación sin obligar a conversation a importar componentes de Tool. Los registros de eventos y vistas neutros respecto al destino son seams de ensamblaje de datos, no registros de componentes paralelos ([decisión](2026-08-09-client-conversation-node-assembly.es.md)).

**Direccionamiento de ámbitos**: replica el modismo de agent-scope del host: los servicios son singletons de raíz cuyos métodos no reciben sessionId — leen la marca de ámbito del llamador (`scopeOf(ctx)`). Dentro de un ámbito de sesión, `ctx.conversation.send('hi', 'queue')` apunta a esa sesión; las llamadas entre sesiones re-apuntan cambiando de ctx (`ctx.sessions.scope(id)!.conversation.send(...)`); llamar a un método con ámbito desde el ctx raíz lanza un error. Los ámbitos de sesión del cliente se acuñan como los agent-scopes del host (una fibra de plugin no-op + una extensión de scope-key), se construyen perezosamente en la primera visualización y solo se derriban cuando la sesión se elimina y queda sin observadores — la muerte de la sesión host por sí sola no derriba un ámbito (se congela en un viewport de solo lectura).

## La capa de objetos de datos (`packages/client/runtime/src/client/sessions/`)

Entran tramas, salen instantáneas, y el ensamblador de Conversation está en medio — sin React (cero imports de React, comprobable con grep):

```
mux/host frames (ConnectionController pump, injected sinks)
        │
        ▼
SessionManager.handleMuxEnvelope / handleHostEnvelope
        │ session frames target existing instances (requested waits buffer)
        ▼
Session.handleMuxEnvelope ──► contiguous Event window
        │                        │ replace / prepend / append
        │                        ▼
        │                ConversationNodeAssembler
        │                  Definitions -> Contexts -> view builders
        ▼
Notifier 微任务合批 ──► ConversationSnapshot 缓存 ──uSES──► 组件
```

- **Session** (session.ts): se construye perezosamente y es residente — una vez creada, sigue comiendo tramas en segundo plano, de modo que cambiar de sesión y volver renderiza al instante. Operaciones: `prompt`/`cancel` (paso directo RPC; los fallos caen en `promptError` de la instantánea), `open` (trae la última página del historial, idempotente), `loadOlder` (paginación hacia arriba, protegida contra reentrada), `resync` (reconexión = vaciar la ventana y volver a ejecutar open). Suscripción: `subscribe`/`getSnapshot` (siempre la referencia cacheada) — `implements ObservableSnapshot<ConversationSnapshot>`, con `useSelector = bindSnapshotSelector(this)` adjunto en la construcción, de modo que una Session es directamente una fuente uSES. El despacho de tramas es un solo switch: las tramas `session/event` deduplican por seq (la única clave de dedup), se ponen en buffer mientras open está en vuelo y, si no, se añaden con proyección incremental; open/stitch fusiona el buffer vivo por seq y rellena una vez si `subscribed.lastSeq` supera la cola de la ventana.
- **ConversationSnapshot** (conversation.ts): el contrato de instantánea inmutable de nivel superior. `chat` contiene el `order` estructural, un lector de Nodes con clave estable por identidad, índices de Turn/Step y la línea de tiempo; `nodes`, `partial`, `runningCalls`, `turnTimings` y `turnEnds` son el tramo de compatibilidad para los consumidores de Trajectory no migrados. Las interacciones pendientes, la cola, running, la eliminación, el estado de apertura, la paginación y los errores de prompt siguen siendo hechos de Session. **Disciplina de referencias** (la premisa de memo y uSES): las subestructuras y los valores de Node sin cambios conservan sus referencias; una actualización de negocio reemplaza solo el valor de la clave correspondiente, salvo que cambien su order o su Location. React sigue suscribiéndose a la Session como única fuente observable, mientras que el `useSession(selector)` proporcionado por el framework aísla las actualizaciones agregadas de Node y Location.
- **SessionManager** (manager.ts): clúster de instancias + entrada de tramas + la lista de sesiones. Las tramas con sessionId van solo a las instancias existentes (una emisión mux no debe instanciar todas las sesiones); las tramas `requested` de approval/question son la excepción — nunca caen en el historial, así que se guardan en `pendingBuffers` y se repiten al instanciar.
- **Notifier** (notifier.ts): dos canales elegidos según el origen del cambio. `markDirty()` (por defecto; los cambios impulsados por tramas siempre) hace batching por microtarea — N cambios, una notificación, un re-render; el flush reconstruye la caché de instantáneas antes de notificar. `notifyNow()` (solo ecos directos de gestos del usuario) reconstruye y notifica en el mismo tick — los inputs controlados revierten el DOM y saltan el caret si su eco se difiere a una microtarea. El código impulsado por tramas que use notifyNow colapsa el batching de vuelta a renders por trama; prohibido.
- **ConversationNodeAssembler** (`runtime/src/client/conversation/`): el motor incremental propiedad de la Session ejecuta Definitions registradas de forma independiente sobre los eventos en bruto. `match(event)` selecciona `(kind, id)` sin escanear Contexts; start/update construyen el estado de Definition; las Locations calculadas por el motor llevan el cierre de Turn/Step; las lecturas de Context hacia atrás registran dependencias que reparan prepends posteriores; `buildViewNode(target)` materializa solo los Contexts sucios. El builder de Chat preserva el orden estructural y la identidad de valor por clave, los selectores `useSession` aíslan el consumo, y la publicación de tokens de Assistant se coalesce en un único frame de animación. [La decisión de Conversation Node](2026-08-09-client-conversation-node-assembly.es.md) es dueña del ensamblaje, mientras que [la propiedad de la presentación de Tool](2026-08-08-client-tool-presentation-ownership.es.md) es dueña del renderizado recursivo de Tool.
- **ConnectionController** (en `packages/client/connection`): abre los streams mux/host, los bombea con for-await, se reconecta con backoff exponencial (500ms duplicándose hasta 10s, jitter, ilimitado) tras una valla de generación; los sinks se inyectan en un solo sentido (el Controller no conoce a Session). Reconexión = reconstrucción: `onConnected` → refresco de la lista + resync por sesión abierta. La capa de objetos solo se enfrenta a `IApiClient`; el transporte Web usa HTTP POST para los dos cuadrantes cliente→servidor y [un WebSocket por stream lógico](2026-08-04-websocket-downlink-carrier.es.md) para los dos cuadrantes servidor→cliente, mientras que la familia de clases de cliente sigue siendo territorio de la nota de capas.

## La cara React (`packages/client/ui-renderer`)

El plugin dinámico ui-renderer es dueño del adaptador ctx↔React, de la instalación a nivel de aplicación, del montaje de la raíz y de la proyección del título. Los componentes de negocio reciben hooks enlazados a través de las props de slot y no importan valores del renderizador.

- El motor de stores de instantáneas **vive en el paquete runtime** (zustand vanilla con actualizaciones basadas en borradores, `flush: 'sync'` por defecto con batching `'raf'` opcional, persistencia en localStorage de valor completo opcional, deep freeze en modo dev — todo exportado desde la entrada principal `./client` de `runtime`, sin subrutas): los productos de store son fuentes observables desnudas sin miembros hook. Los plugins llegan al motor solo a través de declaraciones `defineStore`, según [el estándar del sistema de slots](2026-07-22-slot-type-chain-implementation.es.md). ui-renderer compone cada hook en el punto de enlace (`bindSnapshotSelector`, cacheado por fuente) a partir del único contrato de datos que consume React: `ObservableSnapshot<T>` (`getSnapshot`/`subscribe`) — tanto un objeto Session como un store de instantáneas lo satisfacen.
- `bindSnapshotSelector(source)`: enlaza una fuente en un hook selector tipado sobre uSES-con-selector. Las cuatro cláusulas del contrato de uSES se cumplen por construcción: getSnapshot devuelve la referencia cacheada; subscribe es una clausura del momento de enlace (estable en referencia para siempre); el CSR puro no pasa ninguna instantánea de servidor; la igualdad usa `Object.is` por defecto con `shallowEqual` opcional por llamada.
- Protocolo de igualdad en toda la cadena: los productores usan compartición estructural; los consumidores hacen cortocircuito con `Object.is` o `shallowEqual`; `React.memo` en superficial. La comparación profunda está prohibida en todas partes.

## Forma del directorio

Los paquetes de cliente viven bajo `packages/client/*`, con `apps/web` como la aplicación Vite delgada sobre la exportación de arranque del shell. Los paquetes plugin conservan su mitad de navegador bajo `src/client/`; **todos los artefactos de build caen en `lib/`** — la mitad node como `lib/index.js`/`lib/invariant.js`, el bundle de navegador como `lib/client.js` (el preset compartido de cliente de tsdown emite ambos; no hay directorio `dist/`, y `exports["./client"]` apunta a `./lib/client.js`). `ui-slots`, runtime y ui-renderer forman la dirección de infraestructura; los plugins de funcionalidad cooperan a través de servicios y slots en lugar de importar implementaciones de presentación.

Un paquete plugin multidominio además divide su mitad de cliente por los límites de futuros paquetes — ui-conversation es el ejemplo:

```
src/client/
  contract/    shared slot and cross-domain types
  service.ts   cross-domain orchestration
  skeleton/    conversation shell and details host
  conversation-nodes/ independently registered business Definitions and Chat builder
  chat/        ordered conversation view
  input/       composer state machine
  queue/       queued-message presentation
  settings/    conversation settings rows
  apply.ts     cross-domain assembly point
  index.ts     public contract surface
```

Los archivos de implementación de dominio nunca importan un dominio hermano; las superficies compartidas se enrutan por `contract/`. `scripts/verify-client-domain-graph.ts` hace cumplir las capas (contract=0, domains=1, apply/index=2; los imports solo pueden apuntar a niveles ≤ propios; las aristas entre dominios hermanos fallan). La presentación de Tool ya es un paquete aparte, `ui-tool`, y llega a chat y details solo a través de los slots que declara ui-conversation.

## Cómo desarrollar

- **Una funcionalidad de UI nueva** = un paquete plugin nuevo: declara `dsh.client` (+ topología `inject`) en package.json, escribe la mitad de navegador bajo `src/client/` (apply monta servicios/stores y registra slots), mantén la mitad node como un apply vacío salvo que haya lógica de host, y compila con el preset compartido. Añade el plugin a la configuración del host; el manifest y la carga se siguen automáticamente.
- **Un slot nuevo**: consulta [la nota del estándar del sistema de slots](2026-07-22-slot-type-chain-implementation.es.md) — fusiona el contrato en `SlotMap`, decláralo en el `children` de la entrada padre, renderiza a través de la prop `renderSlot` autoinyectada. Nunca exportes componentes globalmente.
- **Consumir un tipo de trama nuevo**: tramas de sesión solo de transporte → el switch de despacho de Session; tramas a nivel de host → la tabla de enrutado del Manager; eventos de negocio de conversación registrados → una Definition más un renderizador de vista con clave, sin rama de negocio en Session.
- **Dónde vive este estado**: los datos de negocio (eventos, streaming, pendientes) → siempre en la capa de objetos; lo que sabe el padre → props del propietario en el sitio de renderSlot; lo privado de un componente (scroll, texto de búsqueda, expansión) → estado del componente; lo compartido entre entradas o que sobrevive a los remontajes (selección, borradores, anchos de panel) → un store declarado por la entrada ([estándar del sistema de slots](2026-07-22-slot-type-chain-implementation.es.md)).
- **Canal de notificación**: impulsado por tramas/asíncrono = batching con `markDirty`; el eco directo de un gesto de usuario cuyo input controlado necesita el mismo tick = `notifyNow`.

## Consecuencias

Los streams de tokens ya no sacuden el árbol de render: los chunks de Assistant actualizan un Context de negocio y publican su Node con clave como máximo una vez por frame de animación; los resultados de selector de filas no relacionadas conservan sus referencias, de modo que esas filas no se re-renderizan. Las funcionalidades de UI cargan, fallan y se deshabilitan como plugins independientes — una entrada de slot que revienta apaga una tarjeta, un bundle fallido falla de forma ruidosa antes de que la UI se encienda. Los costes aceptados: la maquinaria de loader/tabla de módulos es infraestructura a medida que el equipo posee de principio a fin; el arranque de un solo encendido (sin renderizado progresivo) cambia la granularidad del primer pintado por la simplicidad del ensamblaje; y los dos programas de tipos convierten «qué agregado ve este archivo» en una pregunta que los desarrolladores a veces tienen que responder.

## Alternativas consideradas

| Rechazada | Razón en una línea |
|---|---|
| Un único bundle SPA enlazado estáticamente | Los plugins deben poder componerse por el host en tiempo de ejecución (dirigidos por configuración); un monolito vuelve a acoplar cada funcionalidad de UI a un único build |
| Globals de window / import maps para las dependencias compartidas | La tabla de require con DI mantiene la compartición explícita, con fallos ruidosos e intercambiable; los globals filtran identidad y versión en silencio |
| Datos de negocio en slices de zustand | La ventana de eventos/acumulador es una máquina de estados de comportamiento, no un slice plano; la capa de objetos mantiene la granularidad de instantáneas y el batching bajo control |
| Un registro de componentes paralelo con claves de string para las filas de Tool | El slot hijo con clave de ui-tool transporta el conjunto de nombres de Tool abierto en tiempo de ejecución a través del único modelo de registro de slots ([disolución de toolview](2026-07-23-toolview-dissolution.es.md)) |
| Arranque progresivo/Suspense en la entrega inicial del cliente web | El arranque de un solo encendido es estrictamente más simple; se conserva la cara de estado por plugin del loader para que el encendido progresivo pueda llegar después sin re-arquitectura |
