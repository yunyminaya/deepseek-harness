# Agent Note: Superficies de negocio de comandos web y ensamblaje (ui-commands / ui-skill / ui-subagent)

Status: implemented

[English](2026-07-25-web-command-surfaces-and-assembly.md) | Español

> Alcance: la caché del directorio de comandos y el dispatch de tres clases (ui-commands), el flujo de selección por popup, las dos fuentes de referencia de skill / subagent, y el enrutamiento de comandos de fixture más la aceptación de ensamblaje (la instantánea del flujo slash). El wire portador vive en la [nota de ámbito de sesión](2026-07-25-web-client-session-scope-and-provide-channel.es.md); los disparadores, el menú y la máquina de entrada viven en la [nota de la máquina de entrada](2026-07-25-web-input-machine-and-slash-pipeline.es.md).

## Problema

El pipeline estaba listo, pero el conocimiento de comandos no tenía lugar de aterrizaje: los `ctx.commands` y `ctx.skills` del lado del host estaban completos mientras que el canal web no tenía capacidad de comandos. La capa de negocio tenía que responder:

- La UI de comandos tiene más de una forma (ejecutar en el acto, abrir un cuadro de selección, rellenar hacia atrás y seguir escribiendo argumentos) — ¿cómo distribuyen los paquetes de negocio con cero cambios de esqueleto;
- Cuándo se obtiene el directorio: tirar en cada apertura de menú es demasiado lento, mientras que una caché residente necesita historias de invalidación y reconexión;
- Las sesiones siempre tienen agent detrás (Session + Agent nacidos en el mismo instante) — ¿con qué dirección honra la superficie de comandos del cliente el directorio efectivo por agent del host;
- Aceptación a nivel de ensamblaje: con las capas separadas, cómo se fija la cadena principal visible al usuario cuando se juntan.

## Decisión

### ui-commands: un `CommandUiRuntime` + un `CommandDirectory` con clave de sesión + un `PopupSelectController` por sesión

- La proyección `ClientSessionContext { sessionId }` se mantiene por sí misma en el contrato ui-input-trigger (types.ts): las sesiones siempre tienen agent detrás, así que la identidad de sesión es toda la proyección de la capacidad de comandos; el wire se direcciona por `{sessionId}` (tanto `command.list` como `command.execute`; el host resuelve el Agent desde la cabecera de la sesión).
- El directorio está compartimentado por `SessionId`, con single-flight por clave más una guarda de época (un pull antiguo nunca sobrescribe estado más nuevo); `commands/changed` invalida en blando cada clave (la instantánea antigua sigue sirviendo mientras el repull corre en segundo plano), `connection/reset` invalida en duro cada clave y recalentiza, Enter espera con fuerza a la clave actual, y un fallo conserva el borrador sin degradación. El precalentamiento cuelga del hook `warm` de la fuente — una vez sobre el roster completo en el nacimiento del ámbito, lo que cubre todo el ciclo de vida de la sesión (la capacidad de sesión es constante desde el nacimiento).
- `register(contribution)` registra comandos de cliente (un descriptor + `available(projection)` + una especificación popupSelect); la síntesis de candidatos = el directorio del host + filtrado de disponibilidad de contribuciones, después el pase de consulta/posición, y un choque de nombres host/contribución falla ruidosamente.
- Las tres clases de comandos derivan de las superficies de registro; los desarrolladores nunca declaran posiciones: un descriptor del host con `input` = **leadingInput** (rellenado `/name ␣` + reclamación, seguir escribiendo argumentos, solo posición inicial); una especificación popupSelect registrada por el cliente = **popupSelect** (la shell oficial de cuadro de selección, el negocio distribuye cero componentes); ninguna = **execute** (ejecutar al seleccionar, cero UI).
- La tabla de decisión del dispatch: el menú puede disparar las tres clases; Espacio reconoce solo leadingInput (la defensa contra disparos accidentales: los efectos secundarios irreversibles mantienen solo puntos de entrada explícitos); Enter ejecuta / abre la shell solo sobre un token desnudo, mientras que leadingInput tolera argumentos finales.
- El popup de `popupFor(actx)`: la búsqueda filtra localmente, la selección es single-flight, la proyección se captura al abrir, onSelect consume el token mediante el evento consume-token solo en el éxito, un fallo se retiene para reintentar, y un cambio de sesión solo lo oculta. La shell del popup es una capa transitoria (nunca en la máquina de estados): el cuadro mantiene el foco, Entrar/↑↓/Escape le pertenecen, y hacer clic fuera del cuadro lo descarta (hacer clic en la textarea también devuelve el foco).

### Fuentes de referencia (viendo solo proyecciones más sus propios closures de apply, en el ctx raíz)

- **ui-skill**: `skill.list({sessionId})` se direcciona por sesión (el host resuelve la raíz del proyecto desde la cabecera de la sesión); la caché del directorio es single-flight con clave por sessionId, precalentada en el nacimiento por el hook `warm` y limpiada por completo en `connection/reset`. Una elección produce un resultado de texto (el texto literal `/name `, la decisión de referencia en texto plano); `lexicon` suministra el roster desde la instantánea asentada de CatalogFetch (`undefined` mientras no esté caliente), y `subscribeLexicon` notifica a los listeners por sesión en el asentamiento y en la invalidación. Sin hook de match (las referencias nunca entran en la adjudicación de comandos). Las referencias de skill cabalgan prompts ordinarios como texto literal (fuera del plano de comandos; tool-skill sin cambios, con el directorio de prefijo de sesión dando la asociación cooperativa).
- **ui-subagent**: los candidatos son cero-RPC (la instantánea de sessions.list filtrada por parentId/running); una elección produce un resultado de texto (el texto literal `@name `); `lexicon` deriva de la misma instantánea y `subscribeLexicon` reenvía el feed de cambios del store de la lista (la representación del lado del modelo espera su workstream de negocio).

### Enrutamiento de comandos de fixture y ensamblaje

- El fixture de conexión añade el enrutamiento de comandos (fixture + fake-api): el banco sin clave puede ejecutar el flujo de comandos completo (directorio, ejecución, selección por popup).
- El ensamblaje de apps/cli monta todos los paquetes nuevos; el mapa de rutas tsconfig / los conjuntos de referencias se completan; los catálogos/docs se regeneran con el wire y los eventos.

### Aceptación a nivel de ensamblaje: la instantánea del flujo slash

`apps/web/tests/slash-flow.snapshot.ts` fija la cadena principal visible al usuario (ensamblada sin clave; los mocks de paquetes no sustituyen al transcript ensamblado): el composer deshabilitado sin sesión → crear un Workspace y entrar en una sesión en blanco ya materializada → elegir el `/echo` leadingInput del menú `/` → el comando se ejecuta pero el bit blank no voltea y la lista sigue mostrando `New Session` → la aceptación satisfactoria del primer prompt ordinario convierte esa misma fila; la misma textarea ligada a la sesión se mantiene a través de blank → activo. `workspace-flow.snapshot.ts` fija por separado la creación/reutilización de filas en blanco, el relleno tras rechazo del primer prompt y — en un cambio de Workspace antes del primer prompt — el borrador moviéndose entre máquinas de entrada con la antigua fila en blanco oculta.

## Alternativas consideradas

| Rechazada | Motivo en una línea |
|---|---|
| Dispatch de prompt en línea (el texto de comando cabalgando el mensaje hacia el host para su parseo) | Confunde los planos de comando y mensaje; que la ejecución de comandos sea independiente de la cola de mensajes es semántica existente del host |
| Un puente que materialice skills como comandos | Las skills tienen su propio directorio; N registros serían un rodeo; la forma de etiqueta evita naturalmente el plano de comandos |
| Un RPC `skill.invoke` | El host no tiene tal operación; las referencias de skill son texto plano que cabalga prompts |
| Un nuevo tipo de referencia ContentBlock | Coste de cadena completa (adaptadores/UI/compactación); el texto como verdad más registros de ocurrencia estructurados basta |
| Paquetes de cliente auto-informando directorios de comandos | El host es la única fuente de verdad; el cliente solo lee descriptores, con `commands-changed` empujando la invalidación |
| El eje discriminante `requires: 'none' \| 'agent'` (un directorio sin agent + consultas de doble dirección) | Con sesiones siempre respaldadas por agent, el comando anfibio no tiene propietario; todo el eje se descarta, para reabrirse ante demanda real |
| Slots dedicados commandresult / commandpanel | Los resultados van por notices; la shell del popup es un overlay interno del esqueleto; las tarjetas de resultados ricas viven en el ledger |
| Un directorio por tipo de agent como fuente de `@` | No existe registro de tipos; la instantánea de sesiones vivas ya lo cubre |
| Una familia de clases PickAction/EnterCommand (productos de elección por herencia de clases) | Los valores de runtime entre paquetes rompen la pureza del bundle del cliente; interfaces de datos puras más métodos de closure son equivalentes |

## Consecuencias

- Distribuir un comando de negocio = un registro del host más un `command.register` del cliente (popupSelect) o cero registros (execute/leadingInput derivan automáticamente), con cero cambios de esqueleto; el coste es que la semántica de las tres clases se concentra en ui-commands, y una hipotética cuarta clase significa cambiarlo.
- La caché residente del directorio más la invalidación por push compran menús de latencia cero y adjudicación fiable de Enter; el coste son tres rutas de invalidación (la trama de cambios, la reconexión, la guarda de época) que necesitan pruebas que las fijen.
- El direccionamiento por sessionId pone el directorio efectivo por agent del host (sombras globales + con ámbito) directamente en el wire, presentándolo el cliente tal cual.
- Huecos conocidos: la shell popupSelect aún no tiene consumidor de negocio distribuido (la selección de modelo y compañía llegan con el trabajo `selectModel` del host en forma de mutación en vivo, sirviendo entonces como plantilla de incorporación); el segundo corte de la cola (operaciones de Inbox por elemento), las tarjetas de resultados ricas y la configurabilidad del roster viven en el ledger esperando sus disparadores.
