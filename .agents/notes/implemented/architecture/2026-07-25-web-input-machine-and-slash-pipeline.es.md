# Agent Note: Máquina de estados de entrada web, slots del composer y el pipeline slash (ui-conversation input / ui-input-trigger)

Status: implemented

[English](2026-07-25-web-input-machine-and-slash-pipeline.md) | Español

> Alcance: la máquina de estados de entrada (la tabla de ocurrencias + la vigilancia del claim + la transacción de envío), el hub/fachada y la orquestación de envío, los tres eventos bail con ámbito para reescrituras de entrada entre plugins, la detección de disparadores `/` y `@` y el pipeline del menú (ui-input-trigger), y el sistema de slots alrededor del composer. Depende del modelo de sctx / provide / session-maybe y de entidad en blanco de la [nota de ámbito de sesión](2026-07-25-web-client-session-scope-and-provide-channel.es.md); el conocimiento de comandos (las tres clases, el directorio, los popups) no se toca aquí — eso es territorio de la [nota de superficies de comandos](2026-07-25-web-command-surfaces-and-assembly.es.md).

## Problema

Dos composers, cada uno con ley propia: el hero (EmptyState, la cadena controlada escribiendo directamente en la Session) y la InputBar en conversación (una textarea controlada simple) — comportamiento, propiedad del borrador y ruta de envío, todo inconsistente. Para traer las tres familias de disparadores — comandos `/`, referencias de skill, referencias `@` — a la superficie de entrada, había que responder:

- Cómo se apilan las tres familias de disparadores, y quién tiene el conocimiento de «comandos» frente a quién se mantiene en cero conocimiento;
- Cómo expresa la caja de entrada el «modo comando» — ¿derivado del texto del borrador o estado explícito? ¿Qué significan retroceso, enter, espacio y pegar una línea entera;
- El envío es una transacción asíncrona (una ida y vuelta RPC) — ¿cómo se defienden el reflujo de resultados obsoletos, el cambio de sesión y la reproducción concurrente de React;
- Cómo se representan los chips de referencia en una textarea simple, y quién posee deshacer / portapapeles / emparejado de pegado / serialización del modelo;
- Cómo logran la inversión de dependencias las reescrituras de entrada entre plugins (relleno de menú, inserción de referencias, consumo de tokens);
- Qué shells de React deben reutilizarse a través de sin sesión → sesión en blanco, y qué cuerpos de entrada de sesión estricta pueden sustituirse.

Restricciones duras: los componentes solo se montan a través de slots; los artefactos de presentación nunca entran en el registro de la sesión; la ruta del teclado es segura para IME en todo momento.

## Decisión

### La máquina de estados de entrada (`InputMachine`)

Una máquina de estados pura, eventos dentro / efectos fuera, con reloj inyectado. Cuatro fases (plain / adjudicating / claimed / submitting). El modo comando **nunca se deriva del borrador**; las rutas de elección lo establecen explícitamente en momentos discretos; el claim se vigila con `draft.startsWith(token)`, con una ruptura por retroceso que lo libera automáticamente; la forma del claim es `{token, hint?}` (hint alimenta el texto fantasma).

La superficie de eventos (`dispatch(ev)` es la única entrada de escritura; una transacción por evento):

- `draft-changed {draft, editRange?}` — el borrador completo de la textarea; editRange estrecha el cómputo de desplazamiento de ocurrencias, con un escaneo compartido de prefijo/sufijo por defecto.
- `newline {selection}` — el salto de línea Ctrl+Enter (no mediante el execCommand del navegador: bajo deshacer autogestionado, una escritura del navegador bifurca dos historiales).
- `begin-command {claim, span}` / `insert-ref {reference, span}` / `consume-token {guard}` — el lado de máquina de los tres eventos bail; CAS de span = igualdad de draftRev.
- `set-invalid {invalidIds}` — el bit de estilo para los resultados de resolución de propietario (no es una transacción).
- `undo` / `redo` — el registro de transacciones autogestionado (un anillo de 100; la escritura de un solo carácter se fusiona dentro de ventanas de reloj inyectadas; un envío con éxito limpia el registro).
- `paste-begin {text, selection, components?, generation?}` — el pegado más los componentes coincidentes por instantánea en caliente sincronizados en una transacción (un Deshacer vuelve a antes del pegado); abre un PasteMatchAttempt.
- `paste-upgrade {attemptId, span, reference}` — una mejora asíncrona de coincidencia como su propia transacción (Deshacer en dos pasos); el attempt sigue siendo el actual, e insertedRange se encoge con cada mejora.
- `invalidate-paste` — gestos que terminan el attempt observados en la capa del DOM (operaciones de cursor/selección y similares).
- `enter {mode}` / `adjudicated` / `adjudication-failed` / `submit-settled` / `release` — el plano de la transacción de envío: un SubmitAttempt (seq + AbortSignal) bloquea el reflujo; el éxito confirma y limpia el borrador; el fallo revierte bajo la guarda de deriva (la instantánea del momento del enter se rellena solo mientras el borrador vivo siga igualándola; si el usuario ha vuelto a escribir, solo salta un notice).

La superficie de efectos (ejecutada por la shell): `adjudicate` (llama a InputTriggerController.adjudicate), `begin-submit` (la transacción claim.submit), `default-sink` (mensajes ordinarios, orquestados por el hub), `notice`.

La tabla de ocurrencias y las tres proyecciones del chip:

- Cada referencia ocupa un `U+FFFC` en el borrador; una entrada de tabla es `{occurrenceId, source, ref, offset, label, clipboardText, invalid?}`; los chips del mismo nombre siguen independientes mediante occurrenceId.
- Cada edición actualiza el borrador y la tabla en una transacción: los rangos se desplazan; un borrado/sustitución que intersecta un placeholder actúa sobre el chip entero.
- El placeholder de un solo carácter hace que la atomicidad del teclado se mantenga casi de forma nativa (el cursor no tiene posición interior; Retroceso / teclas de flecha / extensión con Shift se llevan nativamente el chip entero); un clic de ratón en un chip va al golpe de fondo → setSelectionRange de chip entero.
- La proyección visual = label: el fondo renderiza el chip en el offset del placeholder (el glifo de la textarea es invisible), con invalid tomando el estilo inválido.
- La proyección de portapapeles/persistencia = clipboardText: copiar/cortar expande los placeholders dentro de la selección; el espejo de persistencia del borrador escribe la misma proyección (el chat store siempre contiene texto plano; la semántica de semilla de refresco = seleccionar todo-copiar → reabrir → pegar, con los chips degradando a texto a través de un refresco).
- La proyección del modelo = generada por chip en el envío mediante el `codec.serialize` de la fuente (propiedad de la señal y la guarda de obsoletos del attempt de envío; propietario ausente / fallo / cancelación = no enviar, nunca degradar a `/name`).

### Reescrituras de entrada entre plugins: tres eventos bail con ámbito

El contrato se declara en ui-input-trigger (el fondo de la cadena de dependencias); los productores despachan mediante `sctx.bail(sctx, ...)`, y el único lado consumidor son los tres listeners que el hub cuelga del sctx al construir la shell; devolver `true` ⟺ la máquina pasó las guardas de fase y CAS y reescribió de verdad (emitir el evento ≠ modificación satisfactoria; que Espacio reciba `preventDefault` sigue al valor de retorno):

- `slash/input-begin-command` `{claim, span}` — relleno del claim de comando adjudicado desde una elección de menú / Espacio (despachado por el InputTriggerController).
- `slash/input-insert-reference` `{reference, span}` — inserción del chip de referencia (despachado por el InputTriggerController).
- `slash/input-consume-token` `{guard: span | bare-token}` — consumo del token de comando tras el éxito de negocio (despachado por las superficies de comandos aguas abajo).

Las llamadas que permanecen sin evento (inscripción en el registro → llamada explícita → await): el draft/envío de la propia Input, la adjudicación asíncrona de Enter, el serializador de referencias, el emparejador asíncrono de pegado. El `@mode bail` ha entrado en el parser de JSDoc y en el gate del catálogo de cordis (scripts/jsdoc.ts).

### El pipeline slash (ui-input-trigger: un `InputTriggerService` raíz + un `InputTriggerController` por sesión)

Un pipeline de disparador/menú/elección con cero conocimiento de «comandos»:

- El servicio contiene solo el registro de fuentes (`InputTriggerSource{trigger: '/'|'@', name, order?, candidates, onPick, matchSpace?, matchEnter?}`; (trigger,name) único; el `order` opcional ordena el roster — primero el menor, por defecto 0, los empates mantienen el orden de registro — y ese roster ordenado es a la vez orden de grupo y orden de sondeo) y `sessionOf(sctx)`. Implementar un hook de match ES la declaración de participación en la adjudicación de espacio/enter; el pipeline sondea en orden de roster, la primera respuesta no-undefined gana, y sin reclamante significa el sink por defecto. matchSpace es síncrono (el espacio salta a mitad de pulsación; solo caché caliente); matchEnter es asíncrono (puede esperar el calentamiento propio de la fuente, y un fallo de calentamiento rechaza).
- El controlador contiene el único acierto autoritativo (span incluido; retenido para Espacio tras cerrar el menú), el store de menú por sesión, la generación de obtención de candidatos, el arbitraje de teclado (modo combobox: el foco permanece en la textarea, ↑↓/Enter/Escape se interceptan y todos pasan la guarda de composición IME, con la única excepción de que Shift+Enter va primero incondicionalmente) y la orquestación de elección (resultado → eventos bail autodespachados). `toggleSource(name, syntheticHit)` es la ruta de lanzamiento de chrome: siembra solo esa fuente registrada sobre la selección de la textarea del llamador y publica `launcher = name` hasta cerrar; el seguimiento tecleado ordinario limpia el launcher y restaura el roster completo de disparadores. Ambas rutas renderizan el mismo MenuView y ejecutan la misma cadena `onPick`. Un verbo `dismiss()` respalda el `onDismiss` inyectado de MenuView (un puntero abajo fuera del menú y de la tarjeta del composer circundante cierra el menú; MenuView también localiza los títulos de grupo a través del espacio de locale `slash.menu` y limita su altura al espacio de viewport sobre el composer mediante `useAnchoredMaxHeight` de ui-primitives); en el nacimiento de cada ámbito de sesión ejecuta `warm(projection)` una vez sobre el roster de fuentes — dentro de ese ámbito la proyección contiene solo el sessionId estable, sin transiciones publicadas/capacidad; el disposer del ámbito derriba el controlador.
- Las fronteras de palabras de la detección de disparadores (`user@host` y la `/` de URLs nunca disparan) y los niveles de guarda (plain: `/` en todas partes + `@` en línea / claimed: `/` suprimida, `@` viva / frozen: ninguna) son el núcleo puro congelado.

### hub / fachada: la shell residente y el cuerpo de entrada de sesión estricta

- El hub (registros de disparador/decoración + orquestación de envío) toma los servicios slash/comando como dependencias `ctx.get()` opcionales: sin ui-input-trigger o las superficies de comandos, la entrada sigue enviando y recibiendo con normalidad — degradación elegante.
- Cada Session materializada tiene exactamente un `SessionInputShell` (la fachada), creado y derribado con el ámbito de sesión; sin sesión, no se construye ninguna máquina de entrada. `ConversationRoot` es en sí misma la shell residente `session-maybe`, que contiene HeroShell, el selector de Workspaces, la pila del composer y la trama de respaldo de la cadena. Siempre posee el mismo scrollport y asiento del composer; los outlets separados de cabecera y cuerpo de sesión estricta rellenan esas regiones fijas después de que aparezca una Session.
- La barra del composer es una entrada de slot `session-maybe` renderizada incondicionalmente: sin sesión, la misma InputBar se renderiza inerte (caras de máquina ausentes, prop de propietario `disabled`), y una vez que `connectWorkspace` devuelve una sesión en blanco, la misma instancia cobra vida — el DOM de la textarea sobrevive a la transición sin-sesión → en blanco y a cada volteo de fase posterior; `ConversationRoot`, el Hero y el esqueleto de layout se mantienen en todo momento.
- El criterio de Hero de ConversationRoot es `sessionId === undefined || (composerPhase === 'blank' && (openState === 'open' || summaryBlank === true))`: una Session en blanco probada por resumen permanece como Hero en cualquier estado abierto, mientras que una Session no probada se asienta durante la carga. El primer envío entra en engaging síncronamente, y un fallo conserva el composer y el contexto de error en lugar de caer al Hero en blanco; el bit blank de la barra lateral voltea a false solo después de que un prompt se acepte con éxito.
- El envío se unifica en el defaultSink del hub: tras una limpieza optimista del borrador, va solo por `session.prompt` con `mode:'queue'` (la UI web no tiene entrada de steer; el `mode:'steer'` del wire del host permanece fuera de esta máquina); el relleno ocurre solo cuando falla y el borrador vivo sigue vacío — a un usuario que ha seguido escribiendo nunca se le sobrescribe. No existe materialización de Draft ni transacción de adjunto.
- Cuando el Hero en blanco re-elige el Workspace, la shell llama a `connectWorkspace`; si la sesión objetivo difiere, el borrador no vacío se mueve de la shell actual a la shell objetivo antes de abrir el nuevo id, y la antigua sesión en blanco sobrevive pero ya no es la actual.
- El contrato de dos bits del Notifier: `dirty` (frescura de instantánea, limpiable con un pull de `ensureFresh`) y `notifyPending` (deuda de notificación, limpiada solo por un flush) son mutuamente independientes — un pull no debe tragarse un push, y los suscriptores de push de la capa de objetos (watchTransaction) dependen de esta garantía.

### Referencias en texto plano: resultados de texto y decoración del léxico

Las referencias de skill/@subagent se saltan la cadena de placeholder + identidad de ocurrencia — la decisión de referencia en texto plano: una elección inserta el texto literal `/name ` `@name ` directamente en el borrador, con el visual del chip puramente derivado:

- PickOutcome gana un brazo `{text}`; el nuevo evento bail con ámbito `slash/input-insert-text` `{text, span}` (el mismo contrato que los otros tres: CAS de draftRev, devolver true ⟺ reescritura real); facade.insertText pasa por concatenación de setDraft — cero cambios de máquina.
- Las fuentes reciben un hook opcional `lexicon?(session)`: un roster de nombres en instantánea en caliente síncrona, con `undefined` = datos no calientes — cero decoración, nunca dispara una búsqueda (la ruta de render sigue síncrona y sin efectos secundarios); el hook opcional emparejado `subscribeLexicon?(session, listener)` es el canal de invalidación para los rolls que cambian tras el calentamiento (catálogos que se asientan, hijos que nacen/salen). El controlador agrega los rolls en su store de instantánea `lexicon` (re-sondeando en cada notificación de fuente); las fuentes registradas después del nacimiento del ámbito se calientan y se pliegan mediante la difusión en vivo del controlador del servicio.
- `decorations.scanTextRefs`: un escaneo de fronteras de palabras del borrador (`/name`, `@name` al inicio de línea / tras espacios en blanco; `x/name` nunca acierta) contra el roster; un acierto recibe la marca `.textRef` (un resaltado de rango puro en el fondo, igual que hlToken); una edición que rompe la forma de la coincidencia simplemente desaparece en el siguiente escaneo.
- Enviar es el texto literal (sin más serialización `<skill>`); en el lado de la burbuja, MessageItem decora ambas formas (la etiqueta heredada `<skill>` + los tokens de texto plano).
- La antigua cadena de ocurrencia/pegado/serialización permanece en disco completa, sin borrar (aditiva; el borrado es un corte futuro separado). Reactividad de decoración: InputBar se suscribe a la fuente de léxico de la shell (uSES), de modo que un roll que se asienta después del precalentamiento de nacimiento del ámbito enciende los tokens de borrador existentes sin ninguna interacción de menú ni re-render no relacionado.

### Contribuciones provide por sesión y la superficie de teclado privada

- ui-conversation (el hub que también actúa de contribuidor) suministra a través de `sessions.provide` el hook `'input'` (estado de máquina + el overlay de cola) más la prop `inputActions` (`setDraft`/`submit`, callbacks void estables).
- La frontera público/privado: el provide público lleva solo miembros de vocabulario de React; la superficie de comandos de teclado/DOM (track/arbitrate/space/undo/redo/paste/dismissPopup/bindMirror — valores de retorno síncronos, semántica de disposer) es exclusiva de InputBar, pasada privadamente dentro del paquete a través del propio inject de la entrada de InputBar, nunca saliendo del límite del plugin.

### El sistema de slots

`conversation` es en sí misma session-maybe; su contenido de sesión y los slots de entrada del composer son de sesión estricta, mientras que el selector de Workspaces del Hero permanece en la raíz. El registro raíz renderiza el outlet de cabecera sobre su scrollport residente y el outlet de cuerpo dentro de él, antes del asiento residente del composer. Los slots hijos están todos declarados por el registro de conversación de ui-conversation:

- `conversation.session.header` (único) — breadcrumb, pestañas de vista y acciones de cabecera de sesión estricta sobre el scrollport residente.
- `conversation.session` (único) — el anillo de vistas y el espejo de borrador de sesión estricta dentro del scrollport residente. Cabecera y cuerpo comparten el mismo chat store con ámbito de sesión; cada uno se reconstruye cuando cambia el id de sesión.
- `conversation.composer.bar` (único) — el slot para la propia InputBar: la InputBar es una entrada de slot real (auto-registrada en su propio slot) y el contenido del fallback de la cadena del composer; no es una entrada de cadena — la elección única de la cadena la desmontaría en una toma de control, rompiendo la supervivencia del DOM de la textarea.
- `conversation.input.overlay` — el ancla de overlay flotante dentro de la tarjeta de entrada; el inject de los registrantes resuelve el controlador por sesión de cada uno mediante el sessionId del slot.
- `conversation.input.dock` — la franja apilada sobre la entrada (la lista de cola de solo lectura de QueueDock aterriza aquí), ordenada por `order`.
- `conversation.composer.dock` — la banda de estadísticas en el borde superior del composer.
- `conversation.input.left` / `conversation.input.right` — las regiones izquierda y derecha de la fila de herramientas.
- `conversation.input.plan` / `conversation.input.model` (único) — los dos asientos de control con nombre de la fila de herramientas; la barra pasa solo `locked` (props de propietario), cada uno permanece vacío hasta que su plugin propietario se registra, sin fallback de placeholder. El asiento plan permanece vacío mientras esté inactivo porque la fuente Command compartida posee la entrada; un objetivo plan efectivo renderiza el botón de estado `Plan ×` en estado de advertencia, cuya única acción es `/plan off`.
- `conversation.hero.workspace` (ámbito raíz) — el selector de Workspaces compartido por el Hero sin sesión y el Hero en blanco; una elección reutiliza o crea la sesión en blanco objetivo mediante `connectWorkspace`, moviendo el borrador cuando sea necesario antes de cambiar la actual.

### Disciplina de pruebas

Todo el comportamiento de la máquina de estados está cubierto por pruebas unitarias de JS puro (secuencias de eventos dentro, aserción de estado y efectos, cero DOM de navegador); la matriz de interacción se prueba por filas con proyecciones. Este requisito es exactamente lo que forzó la división núcleo puro + shell de servicios.

## Alternativas consideradas

| Rechazada | Motivo en una línea |
|---|---|
| Un estado intermedio ActiveCommand / un registro de modos registerMode / derivar el modo comando del borrador | Los claims se establecen explícitamente por las rutas de elección — sin tabla, sin derivación |
| Cableado directo de objetos bindTarget/bindDraft | Acoplamiento inverso más desemparejamiento entre sesiones del singleton raíz; los eventos bail con ámbito preservan la inversión de dependencias con enrutamiento estructuralmente correcto |
| Un slash/input-apply unificado, o eventarlo todo | Tres payloads independientes cubren las reescrituras entre plugins; las rutas asíncronas siguen siendo llamadas explícitas basadas en registros |
| contenteditable / un árbol de texto enriquecido | Compatibilidad pobre; textarea + U+FFFC + la tabla de ocurrencias cubre el contrato de interacción completo |
| Persistencia dual de borrador {text, occurrences} | El espejo escribiendo la proyección de portapapeles añade cero conceptos nuevos; la degradación de chips a través del refresco es aceptable |
| La pila de deshacer nativa de la textarea | Poco fiable bajo escrituras controladas + programáticas; la semántica de deshacer en dos pasos del pegado solo puede ser autogestionada |
| La InputBar recibiendo un haz de 16 callbacks de cableado | La matriz de consumo demostró 11 miembros exclusivos de InputBar y 1 miembro muerto; el canal del kit estándar permite que los componentes obtengan los suyos, con la superficie de teclado pasada privadamente dentro del paquete |
| La adjudicación de Espacio reclamando también comandos de clase execute | La defensa contra disparos accidentales: tras un espacio, toda la línea es un prompt ordinario; los efectos secundarios irreversibles mantienen solo puntos de entrada explícitos |
| Un mecanismo genérico de decoración tokenPattern | Los registros de ocurrencia estructurados sustituyen al escaneo de patrones |
| Un placeholder select residente en la fila de herramientas | Los asientos con nombre permanecen vacíos hasta el registro; un placeholder que choca con la implementación real son dos fuentes de verdad |
| Un toggle Plan on/off siempre visible | La fuente Command compartida ya posee la entrada; un segundo punto de entrada convierte un asiento de estado en chrome de modo redundante |
| Un segundo componente/controlador de menú plus, o un grupo Add/File sobre Command | Duplicaría los candidatos asíncronos, el resaltado de teclado, la retención de foco y el estado de elección; el control plus es solo un launcher filtrado por fuente para el MenuView existente, y este ámbito no tiene capacidad de archivos |
| Todas las referencias a través de chips U+FFFC (la línea que sustituyó la decisión de texto plano) | El texto plano + decoración derivada porta cero estado de identidad; el texto literal ES la proyección del modelo, ahorrando al deshacer/portapapeles cualquier caso especial; la cadena de chips se conserva para escenarios que necesiten atomicidad indivisible |

## Consecuencias

- Una única shell de conversación residente porta sin-sesión/en-blanco/activo: sin sesión → en blanco conserva ConversationRoot, Hero, el selector de Workspaces de ámbito raíz, scrollport, asiento del composer, InputBar y textarea; solo los outlets estrictos de cabecera y cuerpo ganan contenido. La misma sesión en blanco → enganchando/activo conserva también la InputBar y la textarea. EmptyState y la cadena de intención controlada (`sessions.updateIntent`/`updatePendingPrompt`/`workspaces.sendSession`) se eliminan junto con su último consumidor.
- El cero conocimiento de comandos de la superficie de entrada más dependencias opcionales: la entrada pura funciona sin los paquetes de comandos; las referencias `@` y las referencias de skill reutilizan gratis el mismo pipeline de menú/elección. El coste es que la adjudicación de espacio/enter es un protocolo de sondeo por fuente cuyas semánticas de respuesta (síncrono/asíncrono, el significado de undefined) son un contrato congelado.
- El envío transaccionalizado (seq del attempt + guarda de deriva) hace estructuralmente imposibles las tres clases de defecto — reflujo de resultados obsoletos, cambio de sesión, reproducción concurrente — fijadas por las pruebas de matriz.
- Huecos conocidos: la fidelidad de chips a través del refresco (el emparejado de pegado es reutilizable para ello) aún no tiene workstream; la representación del modelo de la referencia de subagente espera su workstream de negocio.
