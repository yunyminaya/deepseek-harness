# Agent Note: Modelo de paridad de ámbito Agent del cliente web y canal de aprovisionamiento (agents/scope / reutilización de blank / provide)

Status: implemented

[English](2026-07-25-web-client-session-scope-and-provide-channel.md) | Español

> Alcance: el ámbito Agent del cliente (actx) y los eventos dirigidos, el modelo de paridad de materialización cliente/host, el bit de sesión en blanco (blank) y su reutilización (`connectWorkspace`), el canal de aprovisionamiento por sesión (`sessions.provide`) y los pequeños del wire del host que portan estas capacidades (la columna `blank` del resumen, el campo de trama `host/session-added` y la trama `host/commands-changed`). La máquina de estados de entrada y el pipeline slash viven en la [nota de la máquina de entrada](2026-07-25-web-input-machine-and-slash-pipeline.es.md); las superficies de negocio de comandos viven en la [nota de superficies de comandos](2026-07-25-web-command-surfaces-and-assembly.es.md).

## Problema

El cliente web tenía una única superficie de sesión global: todos los slots se renderizaban desde el contexto raíz, de modo que los plugins no tenían noción de «qué agent/sesión es la actual»; la copia verdadera del borrador estaba enterrada dentro del objeto Session, dejando a cualquier plugin que quisiera participar en la entrada sin dónde engancharse. Para soportar un sistema de comandos/entrada, la capa de plataforma tenía que responder primero:

- Quién posee el estado de interacción de la sesión (menús, popups, borradores, peticiones en vuelo), y cómo se aíslan estructuralmente dos sesiones;
- Qué es una «sesión nueva» antes de que exista la entidad del host — si el cliente debe forjar una vida independiente para ella;
- Cómo obtienen los componentes de ámbito de sesión sus propios datos de sesión, en lugar de props pasadas capa por capa;
- Qué deja atrás en el lado del host una sesión nueva abandonada por el usuario, y quién la recoge.

Restricciones duras: el host es la única fuente de verdad; todo registro pasa por un disposer de `ctx.effect`; el mecanismo de ámbito coincide con la arquitectura de ámbito Agent del host; visible al modelo ⟺ ya está en el registro de la sesión.

## Decisión

### El modelo de paridad: cliente y host comparten un único eje de estado raíz

El `session.create(workspaceId)` del lado del host produce Session + Agent + cwd en una sola pieza (un bundle atómico, nunca dividido); el lado del cliente es el espejo de ese nacimiento — en el instante en que una fila de sesión entra en el espejo de la lista, el cliente acuña su ámbito Agent (actx + provide + la superficie de entrada completa montada):

- La identidad de la sesión es la forma verdadera del host desde el nacimiento: el sessionId llega mediante la respuesta de `session.create` / la trama `host/session-added`, y cada dirección del lado del cliente (la etiqueta de ámbito, las claves del store de slots, el direccionamiento RPC) usa ese mismo id.
- El momento de materialización = el instante en que el usuario elige un Workspace (cwd asentada): el cliente llama a `session.create({workspaceId})` en el acto y recibe la entidad completa.
- «Nueva sesión sin Workspace elegido» es un **estado de vista puro** (una posición de navegación) que no corresponde a ninguna entidad de sesión/ámbito; hasta la elección, el composer está bloqueado por completo (sin slash, sin texto plano).
- Una «sesión en blanco» es solo una sesión materializada ordinaria cuyo registro sigue vacío; para todo plugin de ámbito Agent del host (goal/plan/skill/…) es indistinguible de cualquier sesión, de modo que slash/plan están vivos de forma natural.

### Ámbito Agent: el actx es el único portador de sesión en el mundo cordis del lado del cliente

El `agents/scope.ts` del runtime coincide con el `dsh-scope` del host en la capa de mecanismo (fibra + etiqueta + filtro; sin import de valores: el paquete del host lleva el merge `Events` de eventos con ámbito, que colisionaría con el merge de Context dentro del programa del cliente):

- `createScope(ctx, key)`: una fibra de plugin sin op más `extend({[kScope]: key, [Context.filter]: …})` — el filtro vive directamente en el actx: los listeners sin etiqueta reciben globalmente, los etiquetados reciben solo su propio ámbito.
- El dispatch son las primitivas de cordis con thisArg = el propio actx: `actx.bail(actx, event, req)` / `actx.emit(actx, event, payload)`.
- `Session.bindScope(actx)`: se empareja exactamente una vez cuando resolve acuña el ámbito (re-enlazar lanza; dropScope desenlaza), reflejando el `Agent.loopCtx` del host — la Session lo usa para despachar sus propios eventos con ámbito. La dirección inversa actx→Session es un salto a través de `sessions.sessionOf(actx)` (reflejando el uso de `agent.session` de los plugins del host).

Tres divergencias deliberadas respecto al dsh-scope del host:

- El filtro vive en el propio actx en lugar de en un portador separado: la capa wrapper del host protege al sujeto Agent de negocio de desviarse de la clave de ámbito (los eventos del host inyectan el propio Agent como primer argumento), mientras que los payloads de eventos del cliente llevan solo un id — no hay sujeto que proteger.
- Las claves se comparan por valor `SessionId` con marca en lugar de por identidad de objeto: en el host, agent.id === id de sesión (1:1 en el mismo eje), la identidad del agent reutiliza directamente la marca `SessionId`, y la identidad de un ámbito del cliente es su id de wire.
- El ámbito del cliente es un ámbito de **identidad Agent**, no un ámbito de objeto vivo: durante una sesión en frío el objeto Agent del host ya está dispuesto mientras el actx del cliente sigue vivo (en vista) — el eje de identidad está en paridad estricta mientras que el caliente/frío de objetos está deliberadamente desincronizado.

La entrega id→ctx se permite solo en tres tipos de lugares (los providers de negocio nunca entregan):

- Fábricas de inyección de slots: el ctx nunca entra en la capa de render; la identidad que el framework de slots entrega a un componente es el sessionId, intercambiado de vuelta a objetos/controladores mediante mapas de servicios.
- Servicios de coordinación raíz autodireccionados: desde el sessionId de una proyección de vuelta al actx mediante `sessions.scope(id)`.
- Listeners raíz sin etiqueta: buscan su propio store por el sessionId del payload.

### Ciclo de vida del ámbito: anclado al espejo de la lista — nacer es entrar en vista, morir es el prune

Las instancias de Session comparten el ciclo de vida del ámbito; la elegibilidad de vivacidad = estar listada por el host (un único criterio, compartido por acuñado y prune):

- Nacimiento = una fila de sesión que entra en la vista del cliente (el pull de la línea base de la lista / el eco local de `create()` / la trama `host/session-added`); un primer resolve perezoso acuña el ámbito (la resolución es una función pura, segura para render).
- Un prune derriba tres cosas a la vez: la instancia de Session, la fibra del ámbito (en cascada por cada consumidor colgado del actx) y el store de slots con clave de sesión. La sesión en escenario (= `list.current`) es la excepción: retirada mientras sigue en el escenario, conserva una vista de solo lectura congelada y solo se derriba cuando el escenario se aleja.
- Reabrir = reconstruir perezosamente la instancia + `open()` trayendo el historial (el registro de sesión del host es la verdad durable).
- TODO restante: las tramas de aprobación/pregunta nunca entran en el historial y no se pueden recuperar a través de un prune (los pendingBuffers de nivel de manager cubren solo la ventana nunca instanciada).

### El bit blank: la proyección visible de la sesión vacía, su conversión y su reutilización

Una sesión «materializada pero sin primer prompt» se gobierna por el bit `blank` derivado del resumen (una columna derivada, no un campo de cabecera; SessionHeader permanece inmutable):

- El criterio del host: `session.events.length === 0` (cero eventos de registro = aún ningún mensaje de usuario). Una sesión viva lee `summarize()` directamente de la memoria; una sesión en frío es siempre `false` — el contrato de creación perezosa garantiza que una sesión nunca añadida no entra en absoluto en `persistence.list()` (se verifica que los backends JSONL y SQLite son realmente perezosos), de modo que blank nunca toca el disco.
- El wire lo porta en dos lugares: la columna `SessionSummary.blank` requerida y el campo `blank` requerido en la trama `host/session-added` (siempre true en la creación, lo que permite a otras pestañas entrar en el mismo estado de sesión en blanco en sus espejos).
- El espejo del cliente solo baja, nunca sube (monótono), volteado desde tres fuentes, todas reutilizando señales de wire existentes:
  - La propia pestaña del emisor: la **respuesta satisfactoria** al primer `prompt()` voltea a false (la aceptación demuestra que el user/message ya está en el registro del host — este volteo es confirmación, no optimismo; `onEngaged` actualiza síncronamente el espejo de la lista, convirtiendo en su sitio la fila actual de `New Session` en un título ordinario, sin añadir fila a la lista). Un primer prompt rechazado mantiene la sesión en blanco: alineado con la autoridad del host, se sigue mostrando como `New Session`, conservando su elegibilidad de reutilización de connectWorkspace mientras siga siendo miembro de un Workspace.
  - Otras pestañas: la trama `host/session-status (running:true)` lo voltea — una sesión en blanco nunca se ejecuta, así que el primer running implica necesariamente que ya no está en blanco;
  - Alineación de reconexión: el summary.blank de `session.list` es autoritativo, de modo que una pestaña que perdió tramas se alinea de forma natural en su siguiente pull; un blank:true obsoleto nunca puede marcar una sesión convertida como en blanco.
- Disciplina de lista: el store conserva todas las filas; el agrupamiento del navegador de Workspaces, la vista plana, la búsqueda y los recuentos comparten una única proyección visible — cada sesión no en blanco se muestra, mientras que de las sesiones en blanco solo se muestra la que cumple `session.id === sessions.current`, con su título forzado a `New Session`. Tras un cambio de Workspace, la antigua entidad en blanco permanece en el espejo pero queda oculta de la lista mientras se muestra la blank actual del Workspace destino; la superficie visible al usuario mantiene, por tanto, como máximo una fila en blanco globalmente.
- El registro de residuos no requiere GC: tras un refresco, las sesiones en blanco vuelven con el bit intacto y se reutilizan en la siguiente conexión al mismo workspace mientras sigan siendo miembros, de modo que la ruta ordinaria de una pestaña mantiene como máximo una por workspace; tras un reinicio del host, las sesiones en blanco no dejan rastro en disco y simplemente se evaporan; las cáscaras vacías extra de las carreras multi-pestaña solo se convierten en filas ocultas no actuales, digeridas por reutilizaciones posteriores, sin coordinación alguna.

### connectWorkspace: el único punto de entrada de New Session

`workspaces.connectWorkspace(workspaceId): Promise<SessionId>` (propiedad de WorkspaceRuntime — tiene tanto la ruta canónica del workspace como la referencia a sessions):

- El brazo de reutilización: se busca en el espejo de la lista `blank && cwd == workspace.path && sessionIds.includes(id)` — la propia regla de pertenencia del host, nunca solo cwd. Una coincidencia de cwd sin el slot de cuenta (una sesión CLI/TUI nacida en el cwd del host, o un registro borrado/recreado) abriría una sesión que ninguna superficie de agrupamiento puede mostrar bajo este Workspace, así que cae al brazo de creación en su lugar (véase el [arreglo de reutilización por pertenencia](../bug-fix/2026-08-05-workspace-blank-session-reuse-membership.es.md)); un acierto devuelve ese id directamente, sin crear nada.
- El brazo de creación: ante un fallo, `session.create({workspaceId})` devuelve el id nuevo.
- Un workspaceId desconocido falla ruidosamente (nunca crea silenciosamente en otro lugar).
- La garantía de resolución (un contrato para ambos brazos): cuando la promesa se resuelve, el id devuelto ya está en el store de la lista y `sessions.binding(id)` se resuelve síncronamente — `SessionRuntime.create` proyecta la lista síncronamente después del éxito RPC antes de resolver, de modo que un movedor de borradores puede escribir texto en la máquina del nuevo ámbito antes de open, sin esperar un flush de notificadores.
- El llamador toma el id y hace su propio `sessions.open`; enviar el primer prompt es un `session.prompt` ordinario — la sesión ya existe, un fallo es un fallo de prompt ordinario, el texto del borrador sigue en la máquina, y reintentar es simplemente volver a enviar.
- El botón global New Session usa por defecto `recentWorkspaceId`: compara primero el `updatedAt` de la sesión más nueva de cada Workspace, cae al `createdAt` del Workspace cuando no tiene Sessions, y mantiene el orden del host en los empates; solo sin ningún Workspace hace `sessions.clear()` hacia la vista sin sesión. Las acciones de creación dentro de un grupo de Workspaces siguen apuntando explícitamente a ese Workspace.
- En el arranque, el runtime se suscribe a la primera línea base completa: una sesión actual restaurada con éxito se mantiene en su lugar; en caso contrario llama automáticamente a `connectWorkspace(recentWorkspaceId)` y abre la sesión en blanco devuelta. La política se asienta solo una vez; un clear posterior iniciado por el usuario nunca es anulado por la autoselección, y un fallo de conexión espera a la siguiente proyección de línea base para reintentar.
- Re-elegir el Workspace en el Hero en blanco también pasa por `connectWorkspace`; cuando el id objetivo difiere del actual, el borrador no vacío de la máquina de entrada actual se mueve primero al ámbito objetivo y después se hace `sessions.open(nextId)`. La antigua entidad en blanco no se elimina — solo sale de la lista al dejar de ser la actual.

### Aprovisionamiento por sesión: el canal `sessions.provide` del kit estándar

La única vía de aprovisionamiento por la que los componentes de slots de sesión obtienen sus propios datos de sesión. Los plugins declaran un mapa de claves fijo mediante el descriptor estático `sessions.provide({hooks, props, resolve})` (una clave duplicada lanza en el registro); `resolve(binding)` materializa valores para una sesión concreta y los derriba con el ámbito. El `standardKit` de ui-renderer enlaza en un único bucle el compartimento de hooks en hooks de selector `use<Name>` (`observableHook`→uSES, anti-tearing) y pasa el compartimento de props tal cual.

El ámbito de slot es el conjunto cerrado `root | session-maybe | session`:

- `root` recibe solo el kit estándar global, sin identidad de sesión ni aprovisionamiento.
- `session-maybe` sigue a la sesión actual con identidad de ADOPCIÓN (el único comportamiento — no existe el modo mantener-identidad-para-siempre): una encarnación nacida sin sesión conserva su instancia de React a través de la llegada de la PRIMERA sesión (la cáscara en blanco la adopta — sin remount, el DOM sobrevive), y desde entonces se comporta exactamente como una entrada estricta de sesión — cambiar a una sesión distinta remonta, y volver a sin-sesión remonta a una encarnación en blanco nueva que volverá a adoptar. El estado local por sesión de los componentes se limpia, por tanto, por construcción; el estado que deba sobrevivir a un cambio pertenece a fuentes ligadas a la sesión (máquina, store, hooks). Sin sesión, `sessionId`, los resultados de `useSession`/`useInput` y `inputActions` pueden estar todos ausentes. El `SessionMaybeProvider` raíz sin clave impulsa estas actualizaciones suscribiéndose a la proyección atómica `currentProvide` del runtime — los movimientos de selección y los cambios de roster de providers publican a través de la misma fuente, de modo que un cambio de roster bajo un id actual estable republica el bundle montado en lugar de dejar entradas varadas en un esquema de hook/prop obsoleto — mientras que `SessionMaybeProvideInfo` usa el mapa de claves estático para retener la forma completa de hooks/props incluso sin sesión; la contabilidad de adopción por entrada (clave de contador de encarnación) vive en el `SessionMaybeEntry` del renderizador.
- `session` garantiza que `sessionId`, cada fuente de hook y cada prop existen; el error boundary de cada entrada estricta está con clave por `sessionId`, de modo que cambiar de sesión recrea esa entrada y su store de sesión.

`conversation` es la cáscara `session-maybe` residente: `ConversationRoot`, HeroShell, el selector de Workspaces, el scrollport y la pila del composer propiedad de la raíz, y la trama de respaldo de la cadena de overlays conservan sus instancias de React a través del cambio sin-sesión → sesión en blanco. Dos entradas estrictas rellenan regiones fijas sin re-parentar ese árbol: `conversation.session.header` porta breadcrumb/pestañas/acciones sobre el scrollport, mientras que `conversation.session` porta el anillo de vistas y el espejo del borrador dentro de él; ambas comparten el mismo chat store con ámbito de sesión. La barra del composer (`conversation.composer.bar`) es ella misma `session-maybe`: sin sesión sus caras de máquina y acciones de mensaje están inertes, mientras que toda la tarjeta punteada abre el selector de Workspaces ya existente con el puntero y su textarea de solo lectura hace lo mismo con Entrar o Espacio. La misma instancia — textarea incluida — cobra vida cuando aparece una sesión; los slots de entrada restantes siguen siendo `session` estrictos y no despachan nada hasta entonces. La transición blank → enganchando/activo nunca reconstruye la InputBar en un volteo de fase.

- La primera entrada integrada del runtime: el hook `'session'` — el propio `useSession` monta el mismo mecanismo, sin casos especiales.
- Disciplina de concurrencia: el plano de render lee solo del compartimento de hooks (garantía de consistencia de uSES); los callbacks del compartimento de props se usan solo en el espacio de manejadores de eventos; la resolución del descriptor es segura para render (caché idempotente, con el prune cosechando residuos de renders abandonados).
- Los componentes de terceros toman cero dependencias de valores; los tipos son una importación type-only de una línea (fusión de declaraciones en `SessionStandardProps` / `SessionMaybeStandardProps`).

### El espejo de cola de solo lectura

- Semántica de cola: en ejecución no bloquea la entrada; los mensajes ordinarios hacen cola mediante `session.prompt {mode:'queue'}`, y los comandos nunca hacen cola.

### Pequeños del wire del host

- La columna `blank` del resumen y el campo `blank` de la trama `host/session-added` (véase el bit blank más arriba).
- La trama SSE `host/commands-changed` (una señal pura de invalidación); el cliente la enruta a los eventos tipados `commands/changed` y `connection/reset` (difundidos después de que se establezca cada generación de conexión; las cachés derivadas del wire tratan uniformemente el estado anterior como obsoleto). La trama de comandos y su evento tipado de cliente fueron sustituidos después por el reenvío verbatim de `commands/change` a través de `ctx.remote.$on` ([eventos remotos reenviados](2026-08-10-remote-event-delivery.es.md)); `connection/reset` no cambia, y el contrato de invalidación-sin-diff que declara este punto sigue vigente.
- `command.list/execute` y `skill.list` se direccionan uniformemente por `sessionId` (una sesión siempre tiene un Agent; la semántica de reanudación de `agentFor` viene hecha); la narrativa de superficies de comandos vive en la [nota de superficies de comandos](2026-07-25-web-command-surfaces-and-assembly.es.md).
- La forma de la petición `session.create`: workspaceId/cwd como o-o, más un sessionId opcional preasignado por el llamador (un reintento del mismo id y mismo cwd es idempotente; un cwd distinto informa `session-conflict`).

## Alternativas consideradas

| Rechazada | Motivo en una línea |
|---|---|
| Un Intent local del cliente + materializar (CAS publicado / la transacción de adjunto pendingPrompt / la cadena previa a create) | El cliente se ve forzado a simular la primera mitad de vida que el host no tiene, criando una maquinaria de estado enorme — CAS publicado, la transacción de adjunto, publicación parcial |
| IDs reservados por el host (un Map de borradores) | El host solo reconoce un número; la máquina de estados permanece intacta en el cliente |
| Una Session de borrador en el host (una Session sin Agent) | Toda superficie del host que busca el Agent debe bifurcarse para borradores; el core necesitaría una API `attachAgent` más el cwd de cabecera escrito tarde |
| Enlazar un Agent antes del cwd (sin agrupar) | Vuelca el invariante del header.cwd de solo lectura «creado en», más la trampa del producto de efecto secundario del directorio de lanzamiento |
| Pasar el contexto de sesión por React Context | Los plugins deberían tener un único modelo mental entre host y cliente; el mecanismo de ámbito es isomorfo al dsh-scope del host |
| Un portador `scopeTarget` + despachador fusionado (reflejando los `agentEvents` del host) | La capa wrapper del host protege al sujeto Agent de negocio de desviarse de la clave de ámbito; los eventos del cliente no tienen sujeto que proteger — el filtro en el actx más las primitivas de cordis cubren toda necesidad |
| Sesiones sin ctx (una capa de objetos libre de cordis) | Una línea roja nacida solo para que las pruebas unitarias de filtrado eviten importar cordis, al coste de callbacks de contribución de dos saltos más campos públicos mutables; el Agent del host ya tiene loopCtx |
| Instancias de Session residentes (resident-instance) | El registro de sesión del host es la verdad durable; la residencia es mera conveniencia de identidad, y su desalineación con el ciclo de vida del ámbito es una fuente de complejidad |
| Componentes que reciben haces de callbacks de cableado (dos capas inject→props hacia abajo) | El canal del kit estándar permite que los componentes obtengan los suyos; la API pública converge a hooks + props estables |
| Sustituir la vista Hero sin sesión por la Conversación completa de la sesión | Aun con el layout exterior sin cambios, el Hero, el selector y los subárboles del composer remontarían juntos, haciendo saltar toda la región de UI |
| Hacer la InputBar misma `session-maybe` | La máquina de estados de entrada, la superficie de comandos de teclado y las acciones tendrían que aceptar valores ausentes; sustituir solo el cuerpo de entrada deshabilitado mantiene la opcionalidad en el límite de la shell |
| Una trama de conversión dedicada | `session-status(running:true)` implica semánticamente la conversión (una sesión en blanco nunca se ejecuta); añadir una trama compra cero información por un tipo de wire más |

## Consecuencias

- Los plugins ganan contexto de sesión isomorfo al del host: el estado por sesión cuelga del actx y se monta/derriba en una sola pieza con la fibra del ámbito, haciendo las fugas estructuralmente imposibles; el aislamiento de dos sesiones está estructuralmente garantizado por el filtro del ámbito.
- La capa de objetos del cliente converge a un espejo del wire: identidad de sesión, ciclo de vida y adjudicación de capacidades se remiten todas a la entidad del host — el sistema de entrada (la siguiente capa) siempre se enfrenta a una sesión con un Agent real, y los providers como slash/skill se direccionan uniformemente por sessionId directamente.
- El gobierno de la sesión en blanco toma cero mecanismos dedicados: el estado cabalga un único bit derivado, la visibilidad cabalga la proyección unificada de la lista (solo se muestra la blank actual, como `New Session`), la recuperación cabalga el contrato existente de persistencia perezosa (evaporación en el reinicio) y el techo ordinario cabalga la reutilización en el mismo Workspace.
- El coste: la disciplina de entrega id→ctx y la disciplina de concurrencia de provide son convenciones en lugar de algo impuesto por tipos, fijadas por revisión y pruebas. El único eje de estado sigue reteniendo las caras de máquina hasta que existe una Session; la tarjeta residente enruta la activación al selector de Workspaces durante ese intervalo ([decisión](../feature/2026-08-07-workspace-picker-composer-entry.es.md)).
- Huecos conocidos: recuperación de aprobación/pregunta a través del prune (TODO); la selección de modelo vuelve en forma de mutación en vivo (el trío `selectModel` del host está hecho, su consumidor de cliente aún no está construido).
