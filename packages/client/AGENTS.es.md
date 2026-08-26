# AGENTS.md — Pila del cliente web

[English](AGENTS.md) | Español

Reglas para `packages/client/*` (el lado de navegador de la GUI web de dsh) más su entrada de construcción `apps/web`. Complementan las [convenciones](../../AGENTS.es.md#conventions) de todo el repo y las [reglas de paquetes](../README.es.md). Antes de tocar slots, props de componentes, stores o la estructura de plugins, lee el [estándar del sistema de slots](../../.agents/notes/implemented/architecture/2026-07-22-slot-type-chain-implementation.es.md) (el modelo de composición definitivo) y la [nota de arquitectura del cliente web](../../.agents/notes/implemented/architecture/2026-07-19-gui-web-client-architecture.es.md) (cadena de carga, capa de objetos, servicios).

Los paquetes de aquí se nombran con el prefijo de directorio: `@deepseek-ai/dsh-client-<name>`.

## Disciplina de slots y props

El [estándar del sistema de slots](../../.agents/notes/implemented/architecture/2026-07-22-slot-type-chain-implementation.es.md) es dueño del diseño completo; estas son las reglas que no debes violar al escribir o revisar código de cliente:

1. **Una API**: un plugin compone la UI solo a través de `ctx.slots.register({ name, children?, store?, inject? }, Component)`. No hay una llamada separada de definición de slot, ni objeto de cara de whitelist, ni helper de acuñación de caras. Solo el shell renderiza `'root'`.
2. **children = declaración + autorización**: los slots que tu componente renderiza son exactamente las claves del objeto `children` de tu llamada a register (valores de spec: `kind`/`scope`). Renderizar un slot que no declaraste, o declarar uno que declaró otra persona, falla en la carga — no lo soluciones dando la vuelta; el conflicto es el diseño hablando. Los nombres de slot reflejan la ruta de composición: `<domain>.<entry>.<hole>` (p. ej. `'tool.call.toolview'`).
3. **Las props de componente son las cuatro shares, todas derivadas**: `PropsRuntime<K>` (SlotMap: parámetros del owner + `useSession`/`sessionId` en el ámbito de sesión + `useSessions`/`useWorkspaces` globales) & `PropsRenderSlots<S>` (claves de children) & `PropsStore<H>` (factory de store) & la cara de inject. Nunca escribas a mano un miembro que una share ya deriva; nunca re-tipees una share localmente.
4. **Los hooks los hace solo el framework**: `useSession`, `useSessions`, `useWorkspaces`, `useStore`, `renderSlot` son los cinco asientos fijos, más los hooks `use<Name>` que el renderizador enlaza desde las contribuciones de provide y los compartimentos `hooks` de inject. El código de negocio nunca crea un hook o selector como valor de prop — pasa datos planos y callbacks. (Los hooks de comportamiento internos al componente que no se suscriben a nada externo están bien.)
5. **Los datos en vivo tienen exactamente tres canales**: los conoce el padre → props del owner en el sitio de renderSlot; solo los conoce el componente → estado local; compartidos entre entradas o que sobreviven a remounts → un store declarado en register. Los datos derivados son una función pura sobre datos de hooks del framework (`useMemo`), nunca una suscripción propia.
6. **Stores: lee `props.useStore`, escribe `props.actions.*`** — las acciones declaradas son la API de mutación completa. Escribe el store como una factory `createXXXStore()` exportada (los handles a nivel de módulo están prohibidos — singletons de facto); comparte pasando un handle a varios registers dentro de `apply`. El código de producción nunca llama a la factory ni a `.create()` fuera de `apply`; los tests sí (esa es la ruta sancionada sin maquinaria).
7. **inject devuelve datos planos y callbacks** desde el ctx propio del cierre de apply — sin hooks hechos a mano, sin productores de ReactNode, sin objetos de servicio completos. Un hecho reactivo privado del registrante usa el compartimento reservado `hooks` (observables desnudos que el renderizador enlaza a `use<Name>`; los componentes nunca ven las fuentes). El plugin solo puede usar las dependencias nombradas por su declaración `inject`; no hay un ctx más amplio al que llegar.

## Disciplina de lectura reactiva y moneda de contrato

Cómo llegan los datos en vivo al código de render, y qué dominios de UI pueden compartir:

1. **Todo lo que un render lee y que puede cambiar fuera de React llega a través de un hook del framework** (regla 4 arriba). El código de manejadores de eventos puede leer instantáneas en vivo (p. ej. `keyboard.snapshot`); el código de render se suscribe.
2. **Los componentes de negocio no contienen maquinaria de suscripción** — sin `useSyncExternalStore`, sin cableado manual de suscripciones, sin reflejar una instantánea externa en el estado local o en un segundo store. Da a cada hecho reactivo su canal propietario: privado del registrante → el compartimento `hooks` de inject; entre entradas o que sobrevive a remounts → un store declarado; estándar por sesión → `sessions.provide`.
3. **Escalera de acceso a datos** — resuelve las necesidades en este orden: hooks del framework (asientos fijos + `use<Name>` enlazados por provide/inject) → un store declarado (`useStore`/`actions`) → callbacks de inject → cualquier otra cosa es un nuevo punto de extensión del framework y necesita arbitraje del hilo principal.
4. **Los dominios de UI comparten solo datos y callbacks compatibles con JSON.** Las props del owner, los valores inyectados, el estado del store y las contribuciones de provide son datos planos serializables o callbacks sobre tales datos. El compartimento `hooks` inyectado es el único lugar para observables desnudos, y los componentes nunca reciben esas fuentes directamente. Encamina el contenido ReactNode a través de un slot; no añadas props del owner con valor ReactNode ni miembros inyectados (los campos `accessory`/`overlay`/`leftItems`/`rightItems` existentes del compositor permanecen hasta que se muevan a slots).
5. **Una fuente observable mantiene dos identidades estables**: el objeto fuente en sí (el binding de hook se cachea por fuente) y su instantánea entre cambios (`getSnapshot` devuelve la misma referencia hasta que el hecho se mueve).
6. **Quien reconstruye un valor publicado lo vuelve a publicar a través de la misma fuente en el mismo paso**, y una ruta de registro que puede ejecutarse después de que existan consumidores notifica a los consumidores en vivo como parte del registro.

## Disciplina de export (paquetes de plugin de cliente)

El entrypoint `/client` de un paquete de plugin de UI es su API pública de navegador, no un barrel de conveniencia. Tres reglas se aplican a todo el paquete (no las replantees como comentarios por archivo):

1. **Un plugin de UI no exporta valores más allá de lo que la carga de cordis necesita** — `apply` / `inject` (y `Config` donde esté), más las factories de store consumidas solo como tipo por los componentes (`ReturnType<typeof createXXXStore>`). Los tipos compartidos (datos del owner, valores inyectados, alias de props compuestos) también pueden exportarse. Los componentes de implementación, los helpers puros, las constantes y los handles de store permanecen internos. Añadir cualquier export de valor nuevo requiere el visto bueno del usuario, no un consumidor que coincida.
2. **Los tests del mismo paquete importan internos directamente** — `../src/client/xxx.ts` relativo desde los tests del paquete, o el subpath `./src/*` cuando un spec vive fuera del paquete. Nunca amplíes la API pública para que un test compile.
3. **Los imports entre paquetes de símbolos de otro plugin están prohibidos en principio.** Las rutas sancionadas son el sistema de slots (register/renderSlot) y los servicios de ctx. Si ninguna encaja, detente y escala — no añadas un export para desbloquearte.

## Disciplina de ctx (los componentes nunca ven ctx)

`ctx` pertenece solo al mundo de apply: el cuerpo del plugin y las factories de inject que lo cierran. Los componentes —cada `.tsx` bajo un dominio de funcionalidad— reciben todos los datos y callbacks **a través de las cuatro shares de props**; nunca llaman a un hook que alcance ctx, nunca importan una clase de servicio para tocarla, nunca leen un contexto de React (los componentes de negocio ven cero contextos — `BindingContext` y sus parientes son internos del renderizador). Si un componente necesita algo nuevo, la respuesta es una prop enhebrada desde la fuente de su share (sitio del owner, declaración de store o cara de inject), no un hook.

## Líneas rojas de capas

La pila tiene conocimiento unidireccional, asentado en la [nota de arquitectura del cliente web](../../.agents/notes/implemented/architecture/2026-07-19-gui-web-client-architecture.es.md):

1. **Capa de objetos de datos** (`runtime`, sin React): `ConnectionController` → `SessionManager` → `Session` son dueños de todo el estado de negocio (ventanas de eventos, acumulación de streaming, máquina de reconexión), y el motor de stores de instantáneas (zustand/immer, `defineStore`, `shallowEqual`) vive aquí también — los productos de store son fuentes observables desnudas sin miembros de hook. Cero imports de React — comprobable con grep.
2. **Maquinaria de render** (`ui-renderer`, plugin dinámico): toda la integración ctx-a-React — renderizador de slots/outlets, `SessionProvider` y el adaptador uSES. Cada hook se compone aquí en el sitio de binding desde fuentes desnudas; el código de negocio de producción no lleva ninguna dependencia de valor de ui-renderer.
3. **Componentes de presentación** (`src/client/` de los paquetes de plugin, props puras): consumibles, se espera que se reescriban por completo. La lógica de negocio no debe filtrarse en ellos; todo llega a través de las cuatro shares de props.

Innegociables entre capas:

- **Los datos de negocio viven en la capa de objetos, nunca en un store.** Los stores declarados por entrada llevan el estado compartido de visualización/interacción (selección, borradores, anchos de panel); las sesiones, los frames y las conexiones permanecen en la capa de objetos.
- **rpcId es estrictamente bidireccional**: el iniciador lo acuña, el respondedor lo repite; las firmas de negocio ven solo `RpcRequest<P>`, la acuñación permanece en la capa del carrier ([nota de capas y protocolo RPC](../../.agents/notes/implemented/architecture/2026-07-19-gui-layering-and-rpc-protocol.es.md)).
- **Disciplina de publicación del notifier**: `notifyNow` es solo el eco directo de un gesto del usuario; las actualizaciones estructurales usan `markDirty` por microtasks en lote, mientras que los chunks visibles de streaming usan `markFrameDirty` acumulativo. Véase `runtime/src/client/sessions/notifier.ts`.
- **La capa web es presentación pura.** Nada de lo que es «cómo dibujar» (vistas de tarjetas de tool, estados de cola) entra en el log de sesión; el host calcula tales datos por frame o los empuja en vivo, y el replay los recalcula — cayendo a la forma genérica cuando no puede. Una entrada nueva *visible para el modelo* sigue requiriendo un evento de sesión (regla de todo el repo).

## Declaración de dependencias

Las secciones npm describen relaciones de instalación y desarrollo; cada cara de construcción decide independientemente qué contiene su artefacto. [`verify-client-packages`](../../scripts/verify-client-packages.ts) comprueba las reglas específicas de cliente y puede reparar la deriva inequívoca de manifests con `--fix`.

1. **Cada paquete de cliente mantiene Cordis en `peerDependencies` y `devDependencies` coincidentes.** Esto incluye los paquetes estáticos porque su cara de Node participa en el mismo contrato de plugin de Cordis.
2. **Un paquete dinámico declara las relaciones dinámicas internas como peer más dev.** Cuentan los imports de producción, re-exports, aumentos de módulo y referencias solo de tipo a un paquete `@deepseek-ai/dsh-*`, así como un paquete nombrado por `dsh.client.inject`. Una dependencia interna solo de test queda solo en dev.
3. **Las entradas de cliente estáticas son solo dev para un consumidor dinámico.** Un paquete sin `dsh.client`, más los módulos de React sembrados por el shell web, pertenece solo a las `devDependencies` del consumidor; nunca pertenece a las `dependencies` o `peerDependencies` de ese paquete dinámico. `packages/client/web` mantiene igualmente Loader, módulos y entradas de UI estáticas como entradas de desarrollo; Cordis permanece peer más dev.
4. **Las librerías instaladas ordinarias permanecen en `dependencies`.** Esto incluye las librerías de implementación privadas empaquetadas en `lib/client.js` y los imports desnudos que quedan en un `lib/index.js` enlazado estáticamente; el host final de Vite, no la construcción de la librería, fusiona y divide estos últimos. Un paquete dinámico nunca pone un paquete `@deepseek-ai/dsh-*` en `dependencies`.
5. **Cada peer tiene un rango de desarrollo coincidente.** Los ciclos de dependencias npm y peer están permitidos; solo el grafo síncrono de peticiones de módulos tiene la regla separada de aciclicidad más abajo.
6. **Las caras de construcción de navegador y Node declaran la externalidad de forma independiente.** Una mitad dinámica de navegador usa la línea base más `dsh.client.external`; una cara enlazada estáticamente externaliza cada specifier desnudo; una cara de Node externaliza sus dependencias de producción ([`tsdown.client.ts`](tsdown.client.ts)). Mover un nombre entre secciones npm no debe cambiar silenciosamente los contenidos del bundle.
7. **Mantén cerrado el payload publicado.** Cada import relativo de runtime y cada asset emitido debe estar cubierto por `files`; el pase publint del repositorio comprueba la vista de publicación exacta.

## Entorno de navegador en build-time

El código de negocio de cliente puede leer estáticamente `process.env.DSH_CLIENT_*`; cada valor referenciado es contenido público del artefacto. El helper compartido de entorno de construcción da a los bundles de Vite y de tsdown dinámico los mismos valores de proceso de construcción, resuelve los nombres sin fijar a `undefined` y no expone ninguna búsqueda o enumeración dinámica. Una construcción raíz completa registra los valores públicos exactos y un digest de todos los artefactos de cliente; los consumidores de release y de artefactos construidos rechazan un registro ausente u obsoleto. Usa configuración de runtime para las elecciones que deben cambiar después de la construcción.

## Módulos compartidos y el grafo de módulos

Una mitad dinámica de navegador o lleva un módulo privadamente o pide la identidad compartida de la tabla de módulos. La línea base del cliente está centralizada en [`web/src/platform.ts`](web/src/platform.ts): `PLATFORM_MODULES` nombra React sembrado por el shell, Cordis y librerías de UI estáticas; `PRELOADED_CLIENT_EXTERNALS` nombra filas dinámicas, actualmente runtime, cuya factory ordinaria `lib/client.js` llega antes del boot del shell.

1. **Los externals de línea base son implícitos para cada bundle dinámico.** No repitas React, Cordis, runtime, `ui-primitives` o `ui-slots` en los manifests de paquete.
2. **`dsh.client.external` añade una petición específica de paquete.** Úsalo solo para un import de valor no de línea base cuya fila dinámica deba materializarse a través de la tabla de módulos. Declara el specifier de import exacto; solo un `/client` final aliasa la fila del paquete.
3. **El silencio significa copia privada.** Las librerías de implementación de terceros ordinarias pueden empaquetarse de forma independiente. Un valor alcanzado solo a través de `import type` se borra y no crea petición.
4. **Una petición tiene dos proveedores posibles.** Un paquete dinámico suministra su propia fila; `PLATFORM_MODULES` suministra una clave exacta de tabla estática. No hay protocolo de alias `dsh.client.provide`.
5. **Valida ambos lados.** El preset de construcción dinámica externaliza la línea base y rechaza los imports de valor de workspace no declarados; [`verify-client-packages`](../../scripts/verify-client-packages.ts) rechaza peticiones malformadas o redundantes, proveedores ausentes y ciclos síncronos de petición.

### El grafo de módulos queda debajo de la DI de cordis

Tres declaraciones se leen como aristas de dependencia y ninguna es intercambiable: el `inject` de servicio de Cordis, el `external` del grafo de módulos y `dsh.client.inject` — las aristas informativas de nombres de paquete de la [lista de verificación de paquete de plugin nuevo](#new-plugin-package-checklist).

| | `inject` de servicio de Cordis | `external` del grafo de módulos |
|---|---|---|
| Unidad | nombre de servicio | specifier de módulo |
| Momento | runtime; el fiber espera | materialización; el `require` entregado a una factory es síncrono y no puede esperar |
| Insatisfecho | permanece PENDING, sin timeout | lanza al momento |
| Quién puede satisfacerlo | cualquier plugin que provea ese servicio, reemplazable | la identidad única de módulo, no reemplazable |
| Ciclos | permitidos | rechazados |

El seam es `loader.internal = modules`: cordis alcanza el código de plugin a través de `EntryTree.import`, así que cada petición de módulo debe ser satisfacible antes de que cordis pueda ordenar la activación por encima. La mitad de nodo de modules emite filas en orden topológico, y `ClientModuleSystem.import`/`prefetch` registran recursivamente las factories de providers dinámicos antes de que sus consumidores se materialicen. Este orden de módulos es independiente de la activación de Cordis: un provider que inyecta servicios puede registrarse primero y activarse último.

`packages/client/web` no es una entrada de Loader. Sus imports estáticos siembran `PLATFORM_MODULES`; las filas dinámicas precargadas por el parser siguen siendo entradas ordinarias de Loader y artefactos ordinarios de `lib/client.js`.

## Disciplina de Conversation Node

- Una funcionalidad de negocio de Chat registra un `ConversationNodeDefinition` y su renderizador claveado `conversation.chat.node`; no añadas su switch de eventos ni su fold a `Session`, `SessionManager` o un dispatcher central incorporado. Sigue el [recetario de Conversation Node](../../docs/cookbook/adding-a-conversation-node.es.md).
- `match(event)` lee solo el evento actual. Cada evento en un Context multi-evento lleva o deriva de forma independiente el mismo id de negocio estable; `update` pliega un Match en State y sigue siendo reproducible de forma determinista por el `seq` del log.
- El hot path de append y los renderizadores nunca escanean la ventana completa de eventos, los Contexts o los Chat Nodes. Acumula en State, publica hechos del mismo Turn/Step a través de `buildLocationData()` y consume datos finales de Node u hooks de Location restringidos.

## Régimen de directorios (paquetes de plugin)

Una funcionalidad de UI = un paquete de plugin (`src/client/` mitad de navegador). Un paquete multi-dominio se divide donde su código podría convertirse luego en paquetes separados — ui-conversation es el ejemplo: `contract/` (la única API compartida), directorios de dominio que nunca importan un dominio hermano, y `apply.ts` como el único punto de ensamblaje entre dominios; `scripts/verify-client-domain-graph.ts` aplica los niveles. El registro pasa por `slots.register` en `apply` — nunca efectos secundarios a nivel de módulo.

## Estilos

[docs/web-styling.md](../../docs/web-styling.es.md) es la autoridad. Los tokens compartidos `--dsw-*` y las hojas globales viven en `ui-theme/src/styles/`; los componentes de funcionalidad consumen alias semánticos a través de CSS Modules y `clsx`, sin colores literales, librería de componentes ni Tailwind. El texto de producto es chino; los comentarios de código, en inglés.

## Pruebas y cobertura

La estructura de pruebas de la GUI (tres niveles, mapa de carriles) está asentada en la [nota del sistema de pruebas de la GUI](../../.agents/notes/implemented/process/2026-07-20-gui-testing-system.es.md); la política de todo el repo en [docs/testing.md](../../docs/testing.es.md).

- Los paquetes fuente de cliente están dentro de la puerta de cobertura del 100 % por archivo (`pnpm run test:coverage`). Los brazos defensivos genuinamente inalcanzables llevan un comentario `/* v8 ignore -- <reason> */` con una razón real, nunca un ignore desnudo.
- Los specs de componentes renderizan con props realistas o un runtime de fixture conducido y afirman comportamiento visible para el usuario, no nombres de clase, internos de hooks o conteos de render.
- El entorno jsdom viene de un pragma por archivo `// @vitest-environment jsdom` en la primera línea del spec; la config compartida permanece en node-env.
- Cada nivel afirma su propia capa. La semántica de la capa de datos pertenece a las suites de runtime y host; los specs de componentes cubren el comportamiento de presentación.

## Antes de hacer push: la escalera de comprobaciones locales

Ejecuta el peldaño más estrecho que cubra lo que tocaste; escala solo cuando la superficie del cambio lo exija.

1. **Cada cambio de código de la GUI** — `pnpm run test:gui` (segundos; sin navegador, sin servidor): las suites de cliente más los paquetes de GUI del lado host. Este es el bucle interno; úsalo tan libremente como un typecheck.
2. **Cualquier cambio que pueda alterar el navegador ensamblado o la salida visible de conversación/UI** (componentes de cliente o texto, `apps/web`, Vite, `dsh-host-webserver`, connection/handler/SSE) — además `DSH_SNAPSHOT=replay pnpm run test:web`: reconstruye el dist del frontend y luego ejecuta el par de smokes de navegador (el caso de host real se auto-omite sin `DEEPSEEK_API_KEY`) más los escenarios e2e reproducidos sin clave. El CI de PR de Linux usa el mismo modo replay de solo lectura. Usa `DSH_SNAPSHOT=refresh` solo después de confirmar un cambio intencional de salida, o `DSH_SNAPSHOT=record` con una clave para re-grabar fixtures.
3. **Antes de un PR** — usa [dsh-pre-push-checks](../../.agents/skills/dsh-pre-push-checks/SKILL.es.md) para seleccionar las comprobaciones estrechas para el diff saliente; no hay un agregado de pre-push de todo el repo.

Si `test:gui` está en rojo por código que no tocaste, ni lo arregles en silencio ni lo ignores: anótalo en tu handoff para que aterrice en el barrido de la siguiente ventana de PR.

## Lista de verificación de paquete de plugin nuevo

Levantar un paquete de plugin nuevo `packages/client/<name>` (ui-workspace es un ejemplo completo; ui-sidebar/ui-user-questions son esqueletos mínimos):

1. **Esqueleto de paquete**: `package.json` (`@deepseek-ai/dsh-client-<name>`, exports `.`/`./invariant`/`./client`/`./src/*`/`./package.json`, manifest `dsh.client`, lista `files`), `tsconfig.json` (extiende `tsconfig.base.client.json`, una entrada `references` por dependencia de workspace más `runtime-diagnostics/invariants`), `tsdown.config.ts` (`clientBundle(id, ['lib/types/index.js', 'lib/types/invariant.js'])`), `src/index.ts` (apply de mitad de nodo vacío), `src/invariant.ts` (compañero con una razón real), `src/css-modules.d.ts` cuando uses CSS Modules, `README.md` con la sección de Model Experience.
2. **Tres superficies de registro, todas requeridas** (faltar cualquiera falla en un punto distinto y posterior): la entrada `references` agregada de `tsconfig.client.json`; una fila `dsh.client` en `packages/bundle/web-app/cordis.patch.yml`; una dependencia en `packages/bundle/web-app/package.json` (los boots de profile resuelven los nombres de fila desnudos a través del fallback curado `$DSH_HOME/profiles/node_modules`, que refleja las dependencias declaradas de la app y de cada bundle — una fila cuyo paquete ningún manifest declara falla al importar). `pnpm-workspace.yaml` ya hace glob de `packages/*/*`.
3. **Semántica del manifest dsh.client**: `platform: 'web'` siempre, y la declaración requiere un export `./client` (el escaneo lanza sin él); `immediately: true` solo para filas de infraestructura de prefetch de etapa uno. `inject` lista aristas de dependencia de nombres de paquete — son **solo informativas** (visualización de preflight, diffing de HMR); no secuencian la activación de entradas ni el orden de apply. El orden de activación es la espera de fiber de inject de Cordis sobre *servicios*, nada más. Una petición `external` no de línea base secuencia su proveedor dinámico antes que el consumidor — véase [módulos compartidos](#shared-modules-and-the-module-graph).
4. **Registrar en el slot de otro paquete**: el orden de apply no está restringido, y un servicio de negocio no es una barrera de declaración. Usa `ctx.slots.inject(name, () => ctx.slots.register(...))`; espera la declaración real, retira la contribución cuando esa declaración colapsa, se re-ejecuta tras la redeclaración y se va con el fiber del plugin llamante. Devuelve un generador que produzca cada registro cuando varias contribuciones deban instalarse y revertirse atómicamente. Un `slots.register` desnudo en un slot no declarado sigue siendo un error; mantén las aristas de servicio solo para los servicios que la contribución realmente lee.
5. Reconstruye el bundle (`pnpm --filter <pkg> bundle`) antes de sondear un servidor `dsh web` en vivo — el registro sirve `lib/client.js`, no fuentes.
6. **Decisiones de declaración**, cada una asentada por [declaración de dependencias](#dependency-declaration) y [módulos compartidos](#shared-modules-and-the-module-graph): si el paquete envía un export `./client`; qué imports de valor no de línea base requieren `dsh.client.external`; qué dependencias de valor dinámico son peer más dev; qué entradas de compilación estáticas son solo dev; y si `files` cubre cada import relativo de runtime y cada asset emitido.

## Lista de verificación de componente nuevo

1. Compón a través de register: añade el slot a `SlotMap`, decláralo en los `children` de su entrada padre y registra tu componente — véase el [estándar del sistema de slots](../../.agents/notes/implemented/architecture/2026-07-22-slot-type-chain-implementation.es.md). No existe otra ruta de composición.
2. Tipa las props como las cuatro shares (`PropsRuntime` & `PropsRenderSlots` & `PropsStore` & cara de inject) — deriva, no escribas a mano. El estado compartido/que sobrevive va en una factory `createXXXStore()` declarada en register; el estado privado del componente permanece local.
3. Los tests de componente alimentan props directamente (`createXXXStore().create()` para los datos del store; stubs planos para los hooks del framework) y afirman comportamiento sin maquinaria de render.
4. Tokens solo en CSS; texto de producto en chino; comentarios en inglés.
5. `pnpm run test:gui` en verde; si el componente cambia la salida ensamblada visible, ejecuta también `DSH_SNAPSHOT=replay pnpm run test:web`.
6. ¿Cambio no trivial? Necesita una Agent Note en el mismo PR (regla de todo el repo) — las notas de GUI de arriba son los precedentes a extender.
