# Agent Note: Ensamblaje de nodo comercial de conversación del cliente e instantáneas de Chat con clave

Status: implemented

[English](2026-08-09-client-conversation-node-assembly.md) | Español

## Problema

Client Session era propietario de ventanas de transporte, estado de conexión e interacciones pendientes mientras también interpretaba eventos de asistente, herramienta, mensaje, comando, compactación, reintento y cola de turno en una extracción de transcripción centralizada. Añadir un nodo comercial requería cambios en los interruptores de sesión, reproducción de historial, índices, cachés y agrupación React; la identidad comercial, la evolución del estado y la presentación final no tenían un propietario independiente.

La ruta anterior también colocaba valores de asistente y herramienta en ejecución fuera del flujo finalizado. Entraban en la lista de nodos ordenados por log solo después del asentamiento, así que su padre React cambiaba y los remontaba incluso cuando el ID comercial y la `key` permanecían estables. Las cargas de historial completo, los prepends anteriores, los anexos en vivo y el streaming de tokens usaban rutas de actualización separadas, dejando la estabilidad de referencia y el recálculo local dependientes de cachés especializados dispersos por el cliente.

Los eventos comerciales también usan diferentes modelos de correlación. Herramienta tiene IDs de llamada, asistente se correlaciona por turno y paso, compactación tiene su propio ciclo de vida y checkpoint, y una inserción de Inbox representa un estado instantáneo en una secuencia. Mantener todas estas distinciones en una sola extracción haría que cada cambio comercial pasara por una búsqueda global e invalidara cachés no relacionadas.

## Decisión

Client Runtime proporciona un motor de ensamblaje de nodo de conversación neutral al objetivo. Los plugins comerciales registran Definiciones de evento, y los plugins de vista registran Constructores de vista por sesión. `ui-conversation` registra las primeras Definiciones integradas y el constructor `chat`; sesión solo envía la ventana de eventos contigua actual al motor y publica su instantánea en lugar de interpretar comercios individuales de conversación.

Esta nota retiene la derivación, validación comercial por comercial, responsabilidades, algoritmos y compensaciones que siguen siendo relevantes después de la implementación.

### Capas de responsabilidad

| Capa | Responsabilidad durable | Explícitamente no es propietario de |
|---|---|---|
| Sesión | Mantener la ventana de eventos contigua, distinguir reemplazo, prepend y anexar, y programar notificaciones de instantánea | Interpretar eventos de herramienta, asistente, compactación u otros comercios |
| Registro de eventos | Retener las Definiciones únicas por `kind` y el único respaldo bajo ciclos de vida de Cordis | Almacenar el contexto o estado de una sesión |
| Ensamblador | Coincidir eventos y mantener contextos, ubicaciones, dependencias y el conjunto sucio de publicación | Interpretar campos de estado comercial u ordenamiento de Chat |
| Definición de nodo | Definir la conversión de un objeto comercial de eventos a estado y nodos de vista finales | Crear contextos, mutar el estado de otro comercio o escanear todos los contextos |
| Constructor de vista | Organizar incrementalmente los nodos finales del objetivo en la instantánea de esa vista | Reinterpretar eventos de sesión sin formato |
| Renderizador React | Renderizar datos propiedad del renderizador mediante el `kind` del nodo final y leer datos comerciales del `ubicación` del nodo actual | Emparejar eventos comerciales, escanear nodos globales o decidir el estado del ciclo de vida comercial |

Las contribuciones al registro son efectos de Cordis. Eliminar una definición causa una reconstrucción de registro de baja frecuencia para sesiones existentes; los eventos comerciales ordinarios no cambian el registro ni reconstruyen cada tipo comercial.

### Contrato general de `ConversationNodeDefinition`

Cada [`ConversationNodeDefinition`](../../../../packages/client/runtime/src/client/contract/conversation.ts) es propietaria independientemente de la conversión de un objeto comercial de eventos a estado y nodos de vista finales. El `kind` de una definición es su nombre único de registro y el espacio de nombres para sus IDs comerciales.

Un evento puede ser reclamado por varias definiciones ordinarias. Por ejemplo, un evento de asistente actualiza tanto el nodo de asistente como la cola de turno, mientras que un evento de reintento actualiza reintento, asistente y la cola de turno. El ensamblador pide el respaldo solo cuando cada definición ordinaria devuelve `null`.

Una definición no contiene datos comerciales mutables entre sesiones. El ensamblador de cada sesión aísla los contextos, estado, dependencias y constructores de vista de esa sesión.

#### `kind`, ID comercial y clave de contexto

El `id` devuelto por `match()` solo necesita ser estable dentro de su definición. Un ID de herramienta puede ser un ID de llamada, un ID de asistente puede ser `turn:step`, y un ID de Inbox puede ser el seq del evento de inserción.

El ensamblador usa `conversationContextKey(kind, id)` para hacer una clave libre de colisiones. Las definiciones que devuelven el mismo `id` todavía no comparten un contexto. El nodo de vista final debe conservar esta clave propiedad del motor y no puede usar `seq` o la posición de renderizado como identidad.

Cada `(kind, id)` tiene como máximo una coincidencia de inicio. Un segundo inicio falla inmediatamente; una definición debe devolver un nuevo ID para representar un nuevo ciclo de vida.

#### `match(event)`

`match(event)` lee solo el `SessionEvent` sin formato actual y devuelve `{ id, role: 'start' | 'update' }` o `null`. No puede acceder a un contexto, historial, un lector, una ubicación o el sobre de vista.

Esta restricción hace que el costo de enrutamiento de un evento dependa solo del número de definiciones registradas. El ensamblador nunca escanea los contextos históricos de una definición para decidir cuál es propietaria de una actualización.

Los eventos de inicio, resultado, recurso, checkpoint y terminal propiedad del comercio deben llevar o implicar directamente el mismo ID. Si un evento no puede producir ese ID, su productor extiende el protocolo de evento; el cliente no adivina desde el "objeto inacabado más cercano."

El `role` describe el ciclo de vida del estado, no la visibilidad. Un inicio puede producir un nodo terminal inmediatamente, mientras que una actualización puede entrar en un contexto pendiente antes de que se cargue su inicio.

#### `ConversationMatch`

Después de una coincidencia exitosa, el ensamblador combina el evento sin formato, la vista de presentación del wire opcional, `role` y la `ubicación` calculada por el motor en un `ConversationMatch` de solo lectura.

Las `matches` de un contexto siempre permanecen en orden ascendente de `seq` de evento, no orden de llegada de red o orden de ingestión de paginación. Si una página de cola proporciona un resultado antes de que una página anterior proporcione su llamada, el orden de coincidencia final todavía coloca la llamada antes del resultado.

La ubicación puede cambiar cuando prepend llena un límite o anexar cierra uno. El ensamblador reemplaza las ubicaciones de solo lectura de las coincidencias afectadas y reproduce el contexto; el código comercial no conserva una copia antigua de ubicación como autoridad.

#### `ConversationNodeContext`

| Campo | Propietario | Semántica visible para la definición |
|---|---|---|
| `key` | Ensamblador | Identidad final estable derivada de `kind + id` |
| `kind` / `id` | Definición + Ensamblador | Espacio de nombres comercial actual e ID comercial |
| `matches` | Ensamblador | Evidencia comercial completa cargada en la ventana actual y ordenada por `seq` |
| `start` | Ensamblador | Coincidencia de inicio única, o `undefined` antes de cargar |
| `state` | Devuelto por definición, retenido por ensamblador | Valor de retorno más reciente de `start`/`update`, o `undefined` antes de la inicialización |
| `current` | Ensamblador | Nodo materializado más reciente o `null` para cada objetivo |

Los campos de contexto de solo lectura no requieren estado comercial profundamente inmutable. Una definición puede devolver un nuevo objeto o mutar el objeto antiguo en el lugar y devolver la misma referencia.

El ensamblador adopta solo el valor devuelto. Devolver `undefined` de `start()` o `update()` es un error de contrato y falla inmediatamente; mutar un objeto sin devolverlo es igualmente inválido.

Una definición puede inspeccionar todas las `matches` para ayudar a construir el estado o un nodo de respaldo, pero no puede añadir ni eliminar coincidencias, reemplazar campos de contexto ni mutar otro contexto.

#### `start(context, match, reader)`

`start()` es el único punto de entrada de inicialización de estado. El ensamblador lo invoca cuando el inicio único aparece por primera vez y adopta su estado devuelto.

Cuando una página anterior cambia el orden de coincidencias, la respuesta de predecesor del lector o los hechos de ubicación, el ensamblador recalcula desde `start()` en lugar de aplicar un parche de dirección inversa al estado antiguo.

El contexto ya puede contener actualizaciones después del inicio cuando `start()` se ejecuta. Después de que `start()` devuelve el estado inicial, el ensamblador todavía invoca `update()` para cada coincidencia posterior al inicio en orden ascendente de log, así que la dirección de ingestión no puede cambiar la extracción final.

El `reader` está disponible solo en `start()`. La inicialización puede leer el contexto activo más cercano de un `kind` especificado estrictamente antes del seq de inicio actual, pero el código comercial no recibe ninguna interfaz general para escanear mapas internos del motor.

Cada nueva invocación de `start()` reemplaza las dependencias de lector registradas por la invocación anterior, así que una definición que cambie su rama de consulta no retiene bordes obsoletos.

#### `reader.previous(kind)`

`reader.previous(kind)` encuentra el contexto más cercano cuyo `candidate.startSeq < current.startSeq` y cuyo estado está inicializado. Nunca devuelve un contexto en el mismo seq, un contexto futuro o un contexto pendiente sin estado.

El resultado contiene la clave, kind, ID, seq de inicio, estado de solo lectura y coincidencias del predecesor. El consumidor interpreta ese estado por sí mismo; el proveedor solo mantiene su estado correctamente y no necesita registrar un método de consulta especializado.

Cada consulta de lector registra una dependencia `{ key, revision, windowGap }`. Un cambio de revisión de un predecesor coincidente reproduce el consumidor; una falta mientras el historial anterior permanece registra un hueco de ventana para un prepend posterior.

Cuando la ventana ya llega al inicio de la sesión, una falta es un `undefined` definitivo. Cuando `hasMore` es true, la definición ve el mismo `undefined`, pero el ensamblador recuerda que el resultado es provisional.

Las dependencias apuntan estrictamente desde inicios anteriores hacia inicios posteriores, así que la reproducción transitiva no puede formar un ciclo temporal. Tanto la cadena de estado instantáneo de Inbox como las lecturas de mensaje de Inbox usan esta restricción.

#### `update(context, match)`

`update()` maneja una coincidencia posterior al inicio que `match()` ya ha enrutado exactamente al `(kind, id)` actual. No decide qué contexto es propietario del evento.

El ensamblador invoca `update()` en orden ascendente de `seq`. Una actualización de cola en vivo puede aplicarse incrementalmente; cualquier inserción no de cola, un inicio recién cargado o una dependencia invalidada causa una reproducción completa desde `start()`.

Cuando no cambian datos comerciales, `update()` devuelve el estado existente. Cuando cambian los datos, puede devolver un reemplazo inmutable o mutar el objeto existente y devolver ese objeto.

El ensamblador no usa la igualdad de referencia de estado para decidir publicación o propagación. Cada actualización aceptada incrementa la revisión del contexto, lo marca como sucio y hace que los consumidores de lector directos o transitivos se reevalúen.

#### `publication(match)`

`publication()` controla cuándo el estado más reciente se materializa como un nodo de vista; no retrasa la ejecución sincrónica de `match()`, `start()` o `update()`.

| Valor devuelto | Comportamiento |
| --- | --- |
| `immediate` | Solicitar una notificación y flush en la microtarea actual |
| `animation-frame` | Coalescer actualizaciones de alta frecuencia en materialización en el siguiente frame |
| `none` | No programar un flush para esta coincidencia; retener su estado y marcador sucio |

Omitir `publication()` significa `immediate`. Los deltas de token de asistente usan `animation-frame`, los contextos de Inbox invisibles usan `none`, y los finales, reproducciones de dependencia y límites de ubicación publican el resultado más reciente a través de una ruta inmediata.

Cada delta dentro de un frame todavía ejecuta actualización. Solo `buildViewNode()`, el trabajo del constructor de vista y la notificación de instantánea React se coalescen; no se pierden tokens.

#### `buildLocationData(context, scope)`

`buildLocationData()` permite que una definición publique un valor de solo lectura derivado de su estado en un paso o turno propiedad del motor sin exponer el estado mutable de otro comercio. El ensamblador siempre materializa `step` antes de `turn`, así que la agregación a nivel de turno puede leer datos de paso actualizados en el mismo flush; llama a `buildViewNode()` solo después de que todos los datos de ubicación estén listos.

Una definición recibe los ámbitos `step` y `turn` por separado y puede devolver un valor o `null` en cualquiera de las fases. Un valor debe identificar las coordenadas exactas de turno/paso y usar el `kind` de la definición como su clave. El ensamblador es propietario del reemplazo y eliminación y rechaza otro contexto que reclame la misma clave de ubicación.

`ConversationStepDataMap` y `ConversationTurnDataMap` usan fusión de declaraciones para restringir claves y valores. Una ubicación expone solo un lector estable `data.get(key)`; los consumidores no pueden obtener el contexto del proveedor ni mutar su estado.

#### `buildViewNode(context, target)`

`buildViewNode()` lee el contexto más reciente durante la publicación y produce directamente el nodo comercial final para el objetivo nombrado. El ensamblador no añade ninguna capa de actividad genérica, candidato de cola o diseño comercial después.

`null` significa que este contexto aún no se ha materializado para el objetivo. En la ruta incremental ordinaria, un contexto que ha devuelto un nodo no nulo no puede más tarde devolver `null`; la ausencia temporal conserva el nodo de la misma clave y usa la representación de visibilidad del objetivo.

El ensamblador verifica `node.key === context.key` y `node.target === target`. El código comercial puede cambiar `anchorSeq`, datos, ubicación o visibilidad, pero no puede cambiar la identidad dentro de un ciclo de vida.

`current` permite que una definición distinga "nunca materializado" de "ya materializado y ahora oculto." La supresión de reintento de asistente lo usa para evitar la retirada ilegal de nodo.

Una definición es propietaria de como máximo un objetivo de vista; las definiciones de solo estado omiten tanto `target` como `buildViewNode()`. Chat y Trajectory registran definiciones comerciales separadas incluso cuando reconocen la misma familia de eventos durables, mientras que el ensamblador compartido suministra las mismas mecánicas de coincidencia, reproducción, ubicación y publicación a ambos objetivos.

#### No `end()` genérico

El motor no expone un ciclo de vida `end()` fijo. Un comercio de evento único se completa en `start()`, un comercio de múltiples eventos registra la finalización en su propia actualización, y un comercio de estado instantáneo de larga duración crea un nuevo contexto para cada evento.

El cierre de paso y turno son hechos de ubicación externos y no mutan el estado comercial. Un cambio de límite reproduce y construye los contextos afectados; cada comercio combina su propio estado de finalización con si su ubicación está cerrada para producir presentación normal, en ejecución o interrumpida.

Los IDs nunca se reutilizan. Los contextos completados permanecen en la ventana actual, proporcionando identidad de render estable y posible evidencia de predecesor para lectores posteriores.
### La ubicación es un hecho de motor de primera clase

[`ConversationLocationIndex`](../../../../packages/client/runtime/src/client/sessions/conversation-location-index.ts) asigna eventos a ubicaciones desde `turn/start`, `step/start`, cargas de turno y paso explícitas, `step/end` y `turn/end`.

La ubicación tiene cuatro formas: `session`, `turn`, `step` y `unresolved`. Turnos y pasos cada uno llevan estado `open`, `closed` o `unknown` más cualquier evento de inicio y fin cargado.

Cada turno y paso también lleva un almacén de datos de ubicación estable por referencia. Una actualización de definición reemplaza solo su clave propia; la misma identidad de almacén puede adquirir nuevos valores mediante anexar o prepend, permitiendo que contextos, constructores de vista y renderizadores React compartan hechos comerciales de nivel de jerarquía resueltos sin copiar ni escanear la matriz global de nodos.

`unresolved` significa que la ventana de historial actual carece de límites precedentes suficientes; no significa a nivel de sesión. Cuando un prepend anterior proporciona esos límites, el índice corrige las ubicaciones de coincidencias y reproduce solo los contextos que poseen esos seqs.

Un evento ordinario anexado solo hereda las coordenadas actuales, mientras que un límite anexado recalcula solo su turno propietario. Prepend reconstruye los hechos de ubicación desde la ventana contigua expandida, pero la lógica de estabilidad por referencia conserva los objetos de turno y paso sin cambios.

El ensamblador también pasa una línea de tiempo estable por referencia a cada constructor de vista. Los comercios no mantienen por separado el orden de turno, listas de paso, valores de último paso o mapas de límite.

## Tres rutas de ventana de eventos

"Escaneo de historial hacia atrás" describe que la UI carga páginas desde la cola más reciente hacia el inicio de la sesión; no significa que una definición ejecute `update()` en reversa. Independientemente del orden del API de historial o la dirección de carga de página, el ensamblador canonicaliza cada ventana actual y cada página nueva en orden ascendente de `seq`.

| Escenario | Rango de entrada | Manejo de contexto y estado | Constructor de vista |
|---|---|---|---|
| Cola inicial o resincronización de historial | Ventana contigua completa actual | Limpiar y reconstruir todos los contextos en orden ascendente de `seq` | `replace()` |
| Cargar una página de historial anterior | Solo eventos nuevos sin duplicar antes de la ventana | Retener identidad de contexto existente, luego añadir coincidencias, ubicaciones, dependencias y reproducciones locales | `apply(upserts)` |
| Anexar en vivo | Un evento de cola contiguo | Coincidir definiciones y actualizar solo los IDs exactos; los límites afectan solo su turno propietario | `apply(upserts)` |

### Cola inicial y escaneo hacia atrás lógico

1. `Session.open()` carga la página de cola más reciente y pasa sus entradas de historial contiguas a `replaceWindow(entries, hasMore)`.
2. `replaceWindow` limpia contextos antiguos, índices de seq de inicio, índices inversos de seq, dependencias de lector y el mapa de entrada.
3. Ordena cada entrada por `seq` de evento y almacena la ventana actual resultante.
4. LocationIndex reconstruye los hechos de turno y paso para esa ventana.
5. El ensamblador visita eventos en orden ascendente e invoca el `match(event)` de cada definición ordinaria.
6. Cada resultado obtiene o crea su contexto `(kind, id)` y entra en la matriz de coincidencias ordenadas de ese contexto.
7. Un inicio ejecuta `start()`; una actualización de cola en estado inicializado ejecuta `update()` directamente.
8. Si la página contiene solo un resultado o recurso y omite su inicio, el ID todavía crea un contexto y recopila coincidencias, mientras el estado permanece `undefined`.
9. Después de coincidir todos los eventos, el ensamblador vuelve a verificar las dependencias de lector así que los estados instantáneos anteriores en la misma ventana se estabilizan antes de que lectores posteriores los lean.
10. Cada contexto se vuelve sucio, y el siguiente flush reconstruye completamente los datos de ubicación en orden paso→turno antes de invocar `buildViewNode()` para cada objetivo.
11. Algunos comercios devuelven `null` sin un inicio; compactación, comando, resultado de herramienta y error de turno pueden construir nodos de respaldo a partir de evidencia de actualización suficiente.
12. Cada constructor de vista recibe el conjunto completo de nodos y la línea de tiempo y establece la instantánea inicial mediante `replace()`.

Esta ruta comienza solo desde la página más reciente en la capa de paginación. El estado dentro de la página siempre calcula hacia adelante, así que la misma ventana no produce resultados comerciales diferentes bajo una dirección de escaneo diferente.

Un contexto sin un inicio no es un error. Es un contenedor de agregación pendiente esperando una página anterior; el `buildViewNode()` de esa definición decide si la evidencia ya lo hace visible.

Si una actualización con el mismo ID es genuinamente anterior al inicio en orden de log, en lugar de simplemente cargada primero, la reproducción falla con un error de protocolo después de que llega el inicio. El orden de llegada puede invertirse; el orden de log comercial no puede.

### Prepend de una página anterior recién cargada

1. `Session.loadOlder()` solicita la página inmediatamente anterior usando el `baseSeq` actual y primero verifica la continuidad entre la cola de la página y la ventana actual.
2. Sesión prepone las matrices de evento y vista sin formato a su propia ventana y pasa solo esa página a `assembler.prepend(entries, hasMore)`.
3. El ensamblador elimina los seqs que se superponen con la ventana actual, luego ordena la página nueva internamente en orden ascendente.
4. Los contextos, estado, nodos actuales e instancias de constructor de vista existentes permanecen intactos.
5. LocationIndex reconstruye hechos sobre la entrada completa expandida y reporta seqs cuya identidad de ubicación realmente cambió.
6. Los contextos que poseen esos seqs actualizan sus ubicaciones de coincidencias y reproducen desde el inicio; los contextos no relacionados no se unen a la reproducción de ubicación.
7. Los eventos nuevos ejecutan los buscadores de definición y entran en contextos existentes o nuevos mediante ID estable.
8. Si la página nueva proporciona el inicio de un contexto pendiente, ese contexto se inicializa desde el inicio y luego aplica cada actualización ya recopilada en orden ascendente.
9. Si la página establece un predecesor de lector más cercano, cambia una revisión de predecesor o elimina un hueco de ventana, el consumidor recalcula desde `start()`.
10. Las dependencias de lector propagan la reproducción hacia seqs de inicio posteriores; ningún evento se aplica en reversa dentro del lote de propagación.
11. Una página vacía que cambia `hasMore` de true a false también vuelve a verificar las dependencias y resuelve un `undefined` provisional a una ausencia definitiva.
12. El flush republica los datos de ubicación de paso/turno y nodos de destino solo para contextos sucios, luego pasa resultados no nulos al constructor de vista `apply()` como `upserts`.

Prepend conserva las claves de contexto existentes y la identidad de nodo actual. Una página puede añadir claves históricas al frente del orden de Chat o corregir la ancla, ubicación, visibilidad o datos de un nodo existente, pero no recrea contextos comerciales no relacionados.

En un cambio estructural, el constructor de Chat recalcula el orden visible y el índice de ubicación secundario desde su almacén con clave. Ese es trabajo de índice de vista; ni vuelve a ejecutar cada definición comercial ni reemplaza valores de nodo sin cambios.

La reparación de hueco de lector es la mayor diferencia algorítmica entre prepend y anexar ordinario. Una página puede añadir nodos históricos visibles y cambiar estados instantáneos de Inbox posteriores y las clasificaciones de mensaje que dependen de ellos.

### Anexar en vivo hacia adelante

1. Sesión acepta solo un evento en vivo inmediatamente después del seq de cola actual; deduplica superposiciones y ejecuta reparación de cola antes de aceptar un hueco.
2. Un evento no límite entra en las coordenadas de turno y paso actuales incrementalmente; un evento límite actualiza los hechos de ubicación para su turno propietario.
3. El ensamblador invoca `match()` una vez en cada definición ordinaria para este evento y no escanea el conjunto de contexto de ninguna definición.
4. Cada resultado exitoso localiza directamente un contexto mediante `(kind, id)`.
5. Un ID nuevo crea un contexto; una actualización de cola normal para un ID existente invoca `update()` una vez.
6. Un inicio o cualquier evidencia insertada antes de la cola usa `replayContext()` completo y retiene la misma semántica de orden hacia adelante.
7. Después de que cambia una revisión de contexto, solo los dependientes de lector registrados se reproducen.
8. El cierre de ubicación actualiza las coincidencias afectadas dentro de su turno propietario y reproduce esos contextos, permitiendo que valores de asistente, herramienta o reintento inacabados adquieran presentación interrumpida o cancelada.
9. El ensamblador toma la urgencia de publicación más alta entre todas las definiciones coincidentes: `immediate` supera a `animation-frame`, que supera a `none`.
10. Sesión enruta el trabajo inmediato al notificador de microtarea y el trabajo de animation-frame al notificador RAF.
11. El flush actualiza los datos de ubicación de paso/turno para contextos sucios, luego invoca `buildViewNode()` y pasa los upserts y la línea de tiempo más reciente de esta transacción a cada constructor de vista.
12. La nueva instantánea React reutiliza claves de contexto estables; la misma herramienta en ejecución→asentada o asistente en streaming→final nunca se mueve entre padres.

El costo de coincidencia comercial de anexar es el recuento de definiciones más los contextos realmente actualizados, independiente del recuento de contextos históricos. Los consumidores de lector y el cierre de ubicación añaden reproducción proporcional a dependencias reales o al turno propietario.

Un cambio estructural de orden de Chat todavía puede reordenar las claves visibles actuales. Una actualización de solo datos reemplaza un nodo con clave y toca su índice de ubicación. La garantía es que los comercios no relacionados no se reextraen y la identidad de nodo sin cambios se conserva, no que cada operación de índice de vista tenga complejidad constante.

### Consistencia entre reemplazar, prepend y anexar

Las tres rutas preservan los mismos invariantes: las coincidencias de contexto están ordenadas por seq, el estado se extrae hacia adelante desde un inicio único, el lector ve solo contextos activos estrictamente precedentes, los datos de ubicación se publican en orden paso→turno, y la clave de nodo depende solo de kind e ID.

`replaceWindow` es el reemplazo completo de baja frecuencia para apertura inicial, resincronización, reparación de hueco y cambios de registro; no implementa la carga ordinaria anterior. Tanto `prepend` como `append` conservan la identidad de constructor y contexto existente.

El tamaño de página, el número de cargas de historial y la coalescencia RAF afectan solo cuándo llega o publica la evidencia. No cambian el estado y nodos finales para una ventana de eventos igual.

## Cómo los comercios integrados usan definiciones

### Coincidencia, ID y estado

| Comercial / `kind` | ID estable | Coincidencia de inicio | Coincidencias de actualización | Estado y lecturas entre contextos |
|---|---|---|---|---|
| Inbox de siguiente turno / `inbox-next-turn` | seq de evento de inserción | Cada `agent/inbox/spliced` dirigido a siguiente turno | Ninguna | Aplicar la inserción actual al estado instantáneo pendiente/reclamado de `reader.previous(ownKind)` |
| Inbox de siguiente paso / `inbox-next-step` | seq de evento de inserción | Cada `agent/inbox/spliced` dirigido a siguiente paso | Ninguna | Construir el mismo estado instantáneo por instrucción; mensaje lee su conjunto reclamado |
| Mensaje / `input-message` | ID de mensaje | `user/message` de superficie de anexar | Ninguna | Usar fuente para un mensaje de contexto, o leer el Inbox de siguiente paso más cercano para distinguir usuario de direccionamiento |
| Asistente / `assistant-step` | `turn:step` | `step/start` | `assistant/chunk`, `assistant/message` final y reintento del mismo paso | Agregar bloques, uso, tiempo de primer token, evidencia final y estado de reintento oculto, luego publicar datos de paso de la misma clave |
| Herramienta / `tool-call` | ID de llamada raíz | `tool/call` raíz | Resultado raíz e inicio/resultado de Code Dispatch | Agregar raíz, hijos y mapa principal; eventos de despacho se enrutan exactamente mediante `rootCallId` |
| Comando / `command` | ID de comando | `command/run` | `command/done` y eventos de checkpoint de ciclo de vida/compactación que llevan un ID de comando fuente | Agregar resultado de comando y evidencia de compactación manual |
| Compactación automática / `compaction` | ID de compactación | `compaction/start` sin un ID de comando fuente | Resumen, fin y checkpoint de reemplazo | Agregar resumen/checkpoint; evidencia de checkpoint suficiente admite respaldo sin un inicio |
| Reintento / `model-retry` | ID de reintento | Reintento 1 `llm/retry` | `llm/retry` posterior y `llm/retry-started` | Agregar intentos de un RetryId y estado programado/iniciado |
| Error de turno / `turn-error` | Número de turno | `turn/start` | `turn/end` de error | Agregar el fallo terminal; el historial de reintento del turno se representa mediante reintento y nunca oculta esta fila |
| Cola de turno / `turn-tail` | Número de turno | `turn/start` | Asistente, reintento, `step/end` y `turn/end` | Retener el fin del turno, leer los datos de asistente de cada paso y publicar datos de turno; usar coincidencias completas para elegir la ancla visual de cola |
| Entregables / `deliverables` | Número de turno | `turn/start` | Llamadas/resultados de herramienta en ese turno | Agregar rutas de mutación exitosas y publicar datos de turno sin producir un nodo de vista |
| Respaldo desconocido / `unknown-surface` | seq de evento | Evento de superficie de anexar no reclamado por ninguna definición ordinaria | Ninguna | Retener tipo/datos sin formato para el respaldo JSON |

### Nodo de Chat y comportamiento de historial/en vivo

| Comercial | `publication()` | Salida de Chat | Comportamiento de historial y tiempo de ejecución |
|---|---|---|---|
| Inbox | `none` | Sin nodo | Recalcular estados instantáneos a lo largo de la cadena de lector cuando prepend proporciona inserciones anteriores |
| Mensaje | Inmediato por defecto | `user`, `steering` o `context` | La reparación de hueco de ventana puede reclasificar la misma clave de mensaje |
| Asistente | RAF para chunks, inmediato para final, ninguno para solo uso/fin | `assistant-step` de la misma clave con estado running/settled/interrupted | Las coincidencias admiten respaldo sin `step/start`; el cierre de ubicación produce presentación de interrupción |
| Herramienta | Inmediato por defecto | Una raíz `tool-call` recursiva que contiene todas las `subCalls` | Una ventana de historial de solo resultado admite respaldo; running→settled conserva su clave |
| Comando | Inmediato por defecto | `command` ordinario o `manual-compaction` integrado | La llegada de checkpoint puede cambiar la ancla sin cambiar la clave de contexto |
| Compactación | Inmediato por defecto | Marcador `compaction` | Un checkpoint puede representar antes del inicio; un inicio anterior desencadena reproducción hacia adelante |
| Reintento | Inmediato por defecto | Un nodo `model-retry` que contiene todos los intentos | Múltiples reintentos actualizan una clave; el cierre de ubicación presenta el último intento programado como cancelado |
| Error de turno | Inmediato por defecto | `turn-error` en fallo terminal | El fin de error admite respaldo sin inicio; la cadena de reintento asentada del turno se representa junto a esto |
| Cola de turno | Solo inmediato para `turn/end`; de lo contrario ninguno | Pie de página `turn-tail` independiente | Calcular cierre/métricas desde los datos de asistente de paso y usar coincidencias del mismo turno para elegir la ancla |
| Entregables | Inmediato por defecto | Sin nodo | El asentamiento de herramienta actualiza incrementalmente los datos de turno; la ranura de extensión de cola de turno lee los archivos producidos |
| Respaldo | Inmediato por defecto | Fila JSON `unknown` | Cubre solo eventos de superficie de anexar; un comercio ordinario que reclamó pero no ha representado un evento no lo duplica |

Inbox demuestra que cada evento puede ser un contexto de estado instantáneo de solo inicio; no cada comercio requiere un par inicio/actualización. El lector vincula cada estado al contexto del mismo tipo anterior en lugar de inventar un ID de ciclo de vida para todo el Inbox.

Reintento, asistente y la cola de turno demuestran reclamaciones independientes en un evento. Cada definición actualiza solo su propio estado y produce su propio nodo de Chat atómico.

Asistente, cola de turno y entregables demuestran composición de datos de ubicación en capas. Asistente escribe datos `assistant-step` para cada paso; la cola de turno deriva datos `turn-tail` de esos valores de paso; entregables mantiene independientemente datos `deliverables` para el mismo turno. Los consumidores leen solo claves fusionadas por declaración, no escanean nodos de otro comercio y no pueden obtener el estado del contexto del proveedor.

Herramienta y comando demuestran agregación de múltiples eventos: el productor suministra un ID compartido, y el contexto construye un árbol o integra compactación internamente en lugar de empujar el emparejamiento al constructor de Chat.

Compactación y resultados de herramienta históricos demuestran respaldo comercial sin un inicio. El motor no impone "sin inicio significa sin representación"; cada definición decide si las coincidencias actuales son suficientes.

Reintento demuestra la división estado/ubicación. Programado e iniciado pertenecen al estado de reintento, mientras que el cierre de paso y turno pertenecen a la ubicación del motor; `buildViewNode()` los combina en presentación cancelada.

El respaldo desconocido demuestra la propiedad del registro: maneja solo eventos de superficie de anexar no reclamados por cada buscador ordinario, y no crea un nodo duplicado solo porque un contexto reclamado devuelve temporalmente `null`.

## Constructor de vista e identidad React

[`ConversationViewRegistry`](../../../../packages/client/runtime/src/client/conversation/view-registry.ts) crea un constructor independiente por sesión para cada objetivo. El registro almacena fábricas y no comparte el ordenamiento ni cachés de ninguna sesión.

El ensamblador llama a `replace({ nodes, timeline })` en reemplazos completos de baja frecuencia y `apply({ upserts, timeline })` para flush ordinarios de prepend/anexar. Los constructores reciben solo nodos de destino finales ya construidos por definiciones.

[`ChatSnapshotBuilder`](../../../../packages/client/ui-conversation/src/client/conversation-nodes/chat-snapshot-builder.ts) mantiene `order`, un almacén de nodos con clave, el índice de ubicaciones de turno/paso `locations`, `timeline` y el segmento `legacy` usado por StatsLine y reflejado en campos públicos de compatibilidad de nivel superior.

Solo una clave nueva o un cambio en `anchorSeq`, visibilidad o identidad de ubicación hace que una actualización de Chat sea estructural. Un cambio de contenido ordinario no reconstruye `order`; el almacén de nodos con clave reemplaza solo el valor de esa clave.

Para un cambio estructural, el constructor calcula el orden visible desde los valores de almacén actuales y reutiliza matrices de índice sin cambios por referencia. Prepend puede añadir claves históricas al frente, anexar puede añadir una clave en la cola o su ancla comercial, y el ordenamiento nunca renombra claves existentes.

[`ChatView`](../../../../packages/client/ui-conversation/src/client/chat/ChatView.tsx) solo atraviesa `order`. Cada [`ChatNodeSeat`](../../../../packages/client/ui-conversation/src/client/chat/ChatNodeSeat.tsx) permanece en la misma lista principal bajo su clave de contexto y despacha el slot con clave `'conversation.chat.node'` por `node.kind`.

[`ChatNodeDataMap`](../../../../packages/client/ui-conversation/src/client/contract/chat-nodes.ts) es un registro de cargas de renderizador fusionado por declaración. Cada módulo comercial registra su propia definición y renderizador con clave; `registerConversationNodes()` y `registerChatNodeRenderers()` solo ensamblan esas contribuciones independientes y no interpretan comercios mediante una unión cerrada o interruptor central. Los integrados siguen viviendo en `ui-conversation`, pero este límite de tipo y registro permite que un comercio se mueva a un paquete independiente sin cambiar el despachador de Chat.

La entrada Chat en `conversation.view` registra `ChatNodeTurnDataInjected` una vez cuando declara el slot hijo `'conversation.chat.node'`. `ChatNodeSeat` pasa solo la clave de nodo estable como `hookContext`; el renderizador de slot combina esa clave con `useSession` de las accesorios estándar oficiales para construir `useTurnData(businessKey)`. Cada renderizador de Chat con clave por lo tanto lee datos fuertemente tipificados de solo lectura desde su propio turno de nodo, y el renderizador de asistente no tiene autoridad de inyección especial.

Los Hooks contextuales a nivel de slot y el `inject.hooks` propiedad de la entrada permanecen como rutas independientes. El último sigue vinculando solo observables propiedad del registro. El primero almacena definiciones por identidad de cara de inyección de slot estable y vincula su fábrica y Hook por cada ocurrencia de render estable. El selector dentro de `useTurnData()` devuelve solo los `turn.data.get(key)` del nodo actual, así que la igualdad de selector filtra publicaciones de sesión no relacionadas.

El `useSession` estándar sigue disponible para cada renderizador de slot con ámbito de sesión. `useTurnData()` estrecha la ruta de lectura común en lugar de actuar como sandbox de permisos. Las estadísticas de ventana completa o índices de objeto arbitrarios todavía pueden leer la instantánea de sesión explícitamente, pero no se modelan como datos de turno del nodo actual.

El streaming de asistente a final y herramienta en ejecución a asentado actualizan solo los datos y las propiedades de ordenamiento necesarias de un Seat. Ya no se mueven desde un contenedor en ejecución de cola hacia el flujo finalizado, así que el asentamiento no restablece el estado local del componente.

Cuando la lógica comercial cambia deliberadamente un nodo materializado a oculto, deja el orden visible y se remonta cuando se vuelve visible. Esta es la retirada explícita de presentación del comercio, distinta de la garantía de Seat estable para running→settled.

El renderizador de herramienta concreto sigue gobernado por la [decisión de propiedad de ui-tool](2026-08-08-client-tool-presentation-ownership.md). La definición de herramienta suministra datos raíz/subllamada recursivos, y `ui-tool` despacha la presentación concreta mediante el slot con clave por nombre de herramienta.

Trajectory registra su propio objetivo y definiciones comerciales contra el mismo ensamblador y ventana de eventos de sesión que Chat. Su constructor de destino preserva el modelo de lectura orientado a escenario sin consumir el segmento heredado del constructor de Chat ni ejecutar una extracción de historial independiente. El constructor de Chat retiene su segmento heredado para StatsLine y los campos públicos de nivel superior; las definiciones específicas de destino no cambian los contratos compartidos de contexto, lector o ubicación.

Las definiciones de destino específicas de Trajectory, el modelo de escenario preservado, la adaptación de direccionamiento, los límites de complejidad y las rutas rápidas de presentación son propiedad de la [decisión de ensamblaje de contexto de conversación de Trajectory](2026-08-11-trajectory-conversation-context-assembly.md).

## Ruta de tiempo de ejecución y renderizado

```text
Ventana de eventos de sesión
  -> ConversationNodeAssembler
       -> Definition.match(event) -> (kind, id, start/update)
       -> Context matches + State + Location
       -> Definition.buildLocationData(step -> turn)
            -> StepLocation.data / TurnLocation.data
       -> Definition.buildViewNode() para su destino declarado
  -> Constructor de vista de destino
       -> chat: ChatSnapshotBuilder -> ChatView -> ChatNodeSeat con clave
       -> trajectory: TrajectorySnapshotBuilder -> stages/layout/table
```

## Verificación

Las pruebas de tiempo de ejecución fijan el registro de ciclo de vida de definición, anexar de ID exacto, recopilación de actualización antes del inicio seguida de reproducción hacia adelante después del inicio, identidad de prepend, reparación de hueco de lector, dependencias transitivas, cierre de ubicación, orden de fase de datos paso→turno, reemplazo de datos de ubicación, cadencia de publicación, retirada ilegal y constructores por destino.

Las pruebas de conversación cubren cada definición de Chat integrada, datos de paso de asistente, datos de turno de cola de turno y entregables, ordenamiento de Chat y compartición estructural, aislamiento de selector, identidad de asistente y herramienta running→settled, despacho de código anidado, direccionamiento, compactación, reintento, interrupción, anclaje de carga anterior y despacho de slot. Las pruebas de Trajectory cubren sus propias definiciones de mensaje, asistente, herramienta, compactación, encabezado de solicitud y límites integradas junto con el modelo de vista orientado a escenario preservado.

Las pruebas de tipo/tiempo de ejecución de slot fijan el inyección común provisto por padre requerido, el tipo `hookContext`, aislamiento de hook entre contextos de nodo, identidad de fábrica/hook estable y la ausencia de rerenders de renderizador comercial para publicaciones de sesión no relacionadas. Las pruebas de hook de observable propiedad de entrada existentes siguen fijando la ruta que no usa una fábrica contextual.

Las instantáneas web ensambladas, pruebas de GUI y escenarios de navegador cubren el gráfico de plugin real. La evidencia del navegador compara asistente streaming→asentado, bash running→asentado y modo código raíz + subllamadas anidadas contra el diseño principal.

Las pruebas de ruta de historial cubren reemplazo completo, prepend no superpuesto, deduplicación de seq superpuestos, convergencia de `hasMore` de página vacía y anexar en vivo. Las ventanas de eventos iguales ingeridas a través de diferentes rutas producen estado comercial y nodos finales iguales.

## Alternativas consideradas

**Mantener la extracción de transcripción centralizada de sesión y extraer solo ayudantes.** Rechazado: la identidad comercial, la reproducción de historial y la invalidación de caché seguirían perteneciendo a un interruptor cerrado; mover funciones no establecería una propiedad independiente.

**Dejar que los renderizadores React escaneen eventos de sesión.** Rechazado: cada vista duplicaría la coincidencia y el estado de ciclo de vida, React se convertiría en autoridad comercial, y la paginación y el streaming recalcularían árboles de componentes no relacionados.

**Pasar nodos globales o índices de ubicación a cada renderizador comercial.** Rechazado: los componentes comerciales escanearían e inferirían su turno/paso actual, y su alcance de suscripción crecería con la ventana. Una definición publica agregados en una ubicación propiedad del motor, y un renderizador lee solo los datos de ubicación de su propio nodo.

**Llamar cada contexto de una definición para cada evento nuevo.** Rechazado: el costo de anexar crecería con el historial, y `update()` combinaría la coincidencia con la conversión. La coincidencia libre de contexto `match(event)` encuentra el ID primero, después de lo cual solo un contexto actualiza.

**Dejar que un buscador de definición lea contextos o escanee historial.** Rechazado: la coincidencia dependería de la dirección de ingestión, las páginas de historial de resultado primero no podrían determinar la propiedad independientemente, y el anexar en vivo regresaría a buscar objetos abiertos.

**Definir una extracción de estado inversa para el escaneo de historial hacia atrás.** Rechazado: cada comercio mantendría dos algoritmos inversos, y la eliminación, la agregación no invertible y las dependencias entre contextos serían difíciles de mantener equivalentes. Las coincidencias ordenadas seguidas de reproducción hacia adelante desde el inicio preservan un significado comercial.

**Hacer que Inbox sea un concepto de motor de primera clase o un contexto de ancho de ventana.** Rechazado: Inbox es estado comercial ordinario y no pertenece al motor genérico. Estado instantáneo por inserción más un lector estrictamente hacia atrás admite prepend, anexar y búsqueda de mensaje juntos.

**Registrar métodos de consulta especializados para lecturas entre comercios.** Rechazado: los consumidores todavía dependerían de APIs de proveedor, y cada nueva relación expandiría una interfaz central. El lector expone el contexto de predecesor de solo lectura de un kind nombrado; el proveedor escribe un estado útil y el consumidor lo interpreta.

**Dejar que un consumidor de datos de ubicación lea el estado del contexto del proveedor directamente.** Rechazado: el consumidor dependería de la forma interna mutable de otro comercio y no podría expresar qué turno/paso posee el valor. Los mapas de datos fusionados por declaración exponen solo el valor de solo lectura seleccionado por el proveedor y coordenadas propiedad del motor.

**Añadir ciclos de vida `end()`, preparados o de restablecimiento de ventana genéricos.** Rechazado: los comercios tienen diferentes condiciones de finalización, y un hueco de paginación no es un ciclo de vida comercial. Los eventos comerciales actualizan estado, el cierre de ubicación desencadena reproducción/construcción, y las dependencias de lector poseen la invalidación de paginación.

**Reutilizar una definición de evento entre Chat y Trajectory bifurcando en `buildViewNode(target)`.** Rechazado: las vistas requieren estado comercial y registros intermedios diferentes, así que una definición compartida haría que cada paquete lleve las condiciones y cargas del otro. Las definiciones propiedad del objetivo mantienen esas elecciones locales mientras comparten los contratos de ingestión y ciclo de vida del ensamblador.

**Añadir un modelo de diseño genérico encima de los nodos comerciales finales.** Rechazado: la actividad, la candidatura de cola y los enums de diseño centralizarían la semántica comercial actual de Chat en el motor de nuevo. Los nodos finales llevan los datos requeridos por el renderizador directamente y comparten solo identidad, ordenamiento y hechos de ubicación.

**Registrar el Hook de datos de turno solo en el renderizador de asistente.** Rechazado: el acceso a ubicación de nodo actual es una capacidad común del slot `'conversation.chat.node'`, no un renderizador comercial. La entrada Chat principal registra el inyección común una vez, y cada renderizador con clave comparte el mismo contrato fuertemente tipado.

**Mantener los valores de asistente o herramienta en ejecución en un contenedor de cola independiente.** Rechazado: el asentamiento los movería entre padres React, y una clave comercial estable no podría evitar el remontaje. Un orden con clave permite cambios de datos y posición sin cambiar la identidad de Seat.

## Consecuencias

Un nodo comercial nuevo puede registrar su buscador, transiciones de estado, datos de ubicación opcionales, nodo de destino final y renderizador localmente sin cambiar el interruptor comercial de sesión. `ChatNodeDataMap` y los mapas de datos de ubicación permiten que un paquete comercial combine datos fuertemente tipados en el contrato; cada evento relacionado todavía debe exponer un ID estable derivable de ese evento solo.

Los paquetes comerciales del host fusionan por declaración los miembros de eventos durables en `@deepseek-ai/dsh-session/types`, mientras que las definiciones de cliente solo importan por tipo las subrutas `/types` del paquete comercial correspondiente. Aumentar la interfaz declarante en lugar de un barril de reexportación da a los programas TypeScript independientes de host y cliente el mismo estrechamiento de evento sin llevar el tiempo de ejecución del host al gráfico de cliente.

La cola inicial, el prepend anterior y el anexar en vivo comparten un conjunto de invariantes de contexto. Inicios faltantes, huecos de lector, ubicaciones desconocidas y deltas de alta frecuencia son estados explícitos del motor y no requieren ninguna caché específica de dirección.

El anexar no escanea contextos históricos; prepend reproduce solo los contextos cuyas coincidencias, ubicaciones o respuestas de lector realmente cambiaron. Un cambio estructural de Chat todavía puede recalcular el orden visible y los índices, pero no vuelve a ejecutar extracciones comerciales no relacionadas ni reemplaza la identidad de nodo sin cambios.

Separar las actualizaciones de estado de la cadencia de publicación extrae cada delta de asistente mientras materializa a lo más una vez por frame de animación. El cierre de paso o turno y los eventos finales pueden publicar inmediatamente el estado más reciente.

Los pasos y turnos se convierten en hogares estables para agregados entre comercios. La cola de turno y los entregables ya no dependen de renderizadores que escanean nodos globales; el `useTurnData()` a nivel de slot estrecha las lecturas comunes al turno del nodo actual y usa la igualdad de selector para aislar actualizaciones no relacionadas.

El costo son nuevos contratos de tiempo de ejecución para registro, ensamblador, datos de ubicación, reproducción de dependencia y constructores por destino, más inyección común propiedad del padre y `hookContext` por ocurrencia en slots de UI. Los autores de definiciones deben entender IDs estables, inicios únicos, reproducción hacia adelante, orden de publicación paso→turno, acceso de lector de solo lectura y la prohibición de retirada de nodo.

`useTurnData()` no revoca la capacidad `useSession` estándar de los renderizadores con ámbito de sesión, así que este límite depende de la guía de API y las pruebas en lugar del aislamiento de capacidades. Los cambios de registro siguen siendo reconstrucciones completas de baja frecuencia; el constructor de Chat todavía mantiene un segmento heredado para StatsLine y los campos públicos de nivel superior, mientras que Trajectory es propietario de definiciones específicas de destino y un constructor sobre la ventana de sesión compartida. Las definiciones integradas permanecen en sus respectivos paquetes de UI, y estos límites de compatibilidad no devuelven la interpretación comercial a sesión.
