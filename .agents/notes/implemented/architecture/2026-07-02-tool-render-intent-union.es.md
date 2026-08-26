# Agent Note: Unión etiquetada de intención de renderizado para la presentación de llamadas de herramienta

Status: implemented

[English](2026-07-02-tool-render-intent-union.md) | Español

> La unión de intención de renderizado sigue vigente para los transportes de UI; su mapeo a ACP (Agent Client Protocol) queda sustituido por [ACP como protocolo solo de automatización](../simplification/2026-07-23-acp-automation-only-protocol.es.md).

## Problema

Una herramienta declara cómo se renderizan sus llamadas en una UI (la tarjeta de llamada de herramienta de un editor) mediante dos callbacks, `presentCall`/`presentResult` en `ToolDefinition`, que retornan `ToolCallPresentation` / `ToolResultPresentation` con una subforma opcional `ToolTerminal`. Estos crecieron de forma incremental hasta convertirse en una **bolsa de campos opcionales**: `title`, `kind`, `rawInput`, `content`, `locations`, `terminal` en la llamada; `title`, `content`, `terminal` en el resultado; `cwd`/`output`/`exitCode`/`signal` en `ToolTerminal`. La división de responsabilidades es confusa:

- Los campos `terminal` del lado de la llamada y del lado del resultado se solapan, y el bridge concilia por llamada un bloque `content` Y un bloque `terminal` Y `rawInput`, cosiéndolos con condicionales ad hoc.
- Qué combinaciones son *válidas* no está escrito: una llamada `terminal` que además fija `content` significa «descripción sobre la tarjeta»; una llamada genérica que fija `terminal` no significa nada pero es representable. El tipo permite el sinsentido.
- No hay forma de expresar la única prestación de herramienta de archivos que más quiere un editor — una **tarjeta de diff** (`{path, oldText, newText}`, que Zed renderiza como diff en línea / vista previa de archivo nuevo). `ToolCallPresentation.content` es el vocabulario `ContentBlock[]` del *LLM* (texto/imagen), así que una herramienta literalmente no puede pedir un diff.

Una propuesta anterior rechazada de colapsar la presentación propiedad de la herramienta aplazó el renderizado rico hasta poder «volver más tarde como una unión etiquetada de intención de renderizado cuando haya al menos dos herramientas reales y dos consumidores reales que validen el vocabulario». Esa barra la cumplen múltiples familias de productores más los consumidores TUI y host/runtime de cliente (Web).

## Decisión

Reemplaza la bolsa de campos opcionales por una **unión discriminada etiquetada con `card`**. Una herramienta declara una intención de renderizado por llamada/resultado; el bridge conmuta según la etiqueta.

```ts ignore-check
type FileLocation = { path: string; line?: number }
type FileDiff = { path: string; oldText: string | null; newText: string } // oldText null ⇒ new file

// presentCall → ToolCallView
type ToolCallView = GenericCallView | TerminalCallView | DiffCallView
interface GenericCallView { card: 'generic'; title: string; kind?: ToolCallKind; rawInput?: unknown; content?: ContentBlock[]; locations?: FileLocation[] }
interface TerminalCallView { card: 'terminal'; title: string; description?: string; cwd?: string }
interface DiffCallView { card: 'diff'; title: string; diffs: FileDiff[]; locations?: FileLocation[] }

// presentResult → ToolResultView
type ToolResultView = GenericResultView | TerminalResultView
interface GenericResultView { card: 'generic'; title?: string; content?: ContentBlock[] }
interface TerminalResultView { card: 'terminal'; title?: string; output?: string; exitCode?: number; signal?: string }
```

`card` es **obligatorio** en cada variante — un discriminante real, no un valor por defecto opcional. El bridge hace `switch (view.card) { case 'generic': … case 'terminal': … case 'diff': … default: assertNever(view) }`. La unión es **cerrada** (según la [convención de exhaustividad del switch](../../../../AGENTS.md.es.md)): una cuarta intención de renderizado (una tabla, un gráfico) necesita código de bridge nuevo para renderizarla de todos modos, así que una variante añadida por un plugin que el bridge descartara en silencio sería peor que un error de compilación. Añadir una variante rompe la compilación en el switch del bridge — exactamente la señal que queremos.

### Por qué una unión etiquetada supera a la bolsa de campos

- **Los estados inválidos dejan de ser representables.** Una tarjeta genérica no puede llevar salida de terminal; una tarjeta de terminal no puede llevar un diff. La bolsa antigua permitía todo esto.
- **Los consumidores conmutan en lugar de coser.** Un brazo por tipo de tarjeta produce exactamente la vista que esa tarjeta necesita, en lugar de conciliar cinco campos opcionales cuyas interacciones no están documentadas.
- **`diff` es una intención de primera clase.** Los write/edit de `dsh-tool-fs` declaran `card:'diff'` con `{path, oldText, newText}`, permitiendo que las UIs capaces rendericen un cambio en línea sin casos especiales por nombre de herramienta.

### Mapeo de productores

- `dsh-tool-fs` read → `generic` (`kind:'read'`, un `location` de seguimiento); write → `diff` (`oldText:null`); edit → `diff` (`oldText:old_string || null`, `newText:new_string ?? ''`). Esto refleja campo por campo los brazos Read/Write/Edit de `toolInfoFromToolUse` de `claude-agent-acp`.
- `dsh-tool-bash` en primer plano → llamada `terminal` + resultado `terminal`; `run_in_background` → `generic`. Los controles genéricos `job_*` son dueños de sus propias tarjetas genéricas.
- `dsh-tool-todo` → `generic`.

### Titularidad del respaldo de terminal

`TerminalResultView` solo lleva `output`/`exitCode`/`signal`. Una UI sin la capacidad de terminal necesita un respaldo de texto delimitado ` ```console `; esa derivación pasa al **bridge** (envuelve `output` en un bloque delimitado en la ruta sin capacidad), en lugar de que la herramienta lo codifique dos veces. Esto mantiene el resultado de la herramienta bash como una única forma estructurada y conserva el comportamiento existente limitado por capacidad byte a byte.

La intención de terminal es solo de visualización. El harness sigue ejecutando el comando a través de su servicio bash, conservando el sandboxing, el saneamiento del entorno, la propiedad de los jobs y el cwd por sesión; una UI proyecta la llamada completada y nunca se convierte en un segundo backend de ejecución.

### Pureza conservada

`presentCall`/`presentResult` siguen siendo funciones puras de `args` (+ el resultado para `presentResult`) — se ejecutan tanto en streaming en vivo COMO en la reproducción del registro de sesión, así que deben ser deterministas respecto a la reproducción. Cada vista se deriva solo de args: el diff de write es de estilo archivo nuevo (`oldText:null`) porque la herramienta no tiene contenido antiguo en el momento de la llamada; el diff de edit es `old_string`→`new_string`.

## Alternativas consideradas

- **Eliminar por completo la presentación propiedad de la herramienta** — la propuesta de colapso rechazada que esta nota sustituye; su propio veredicto aplazaba exactamente a esta unión hasta que existieran dos herramientas reales y dos consumidores reales, y esa barra ya está cumplida.
- **Dejar que una UI ejecute intenciones de terminal** — rechazada porque sortearía la política de bash del harness y los contratos de propiedad, y bifurcaría la ejecución de comandos entre backends. Una tarjeta de terminal describe ejecución propiedad del harness; nunca autoriza la ejecución en el lado del cliente.
- **Una unión extensible por merge** (el patrón `ContentBlockMap`) — rechazada: una intención de renderizado nueva necesita código de bridge nuevo para renderizarla de todos modos, así que una variante añadida por un plugin que el bridge descartara en silencio sería peor que el error de compilación que la unión cerrada provoca en el switch `assertNever` del bridge.
- **Conservar la bolsa de campos opcionales** — el statu quo que disecciona el Problema: estados inválidos representables, interacciones de campos no documentadas y ninguna forma de pedir una tarjeta de diff.

## Consecuencias

Una intención de renderizado nueva es un cambio que rompe la compilación en el switch del bridge — deliberadamente: el código de renderizado debe existir antes que un tipo de tarjeta. Las combinaciones inválidas de tarjeta/campo dejan de ser representables, y la derivación del respaldo de bash vive en el bridge, así que una herramienta retorna una única forma estructurada. La barra para una cuarta tarjeta (una tabla, un gráfico) es escribir su brazo de bridge en el mismo cambio.

## No objetivos

- **Streaming incremental en vivo de `terminal_output_delta`** y **clasificación de comandos** — los seguimientos aplazados propios del Agent Note de renderizado de terminal, intactos aquí.

## Relacionado

- Sustituye el aplazamiento de la propuesta anterior rechazada de colapsar la presentación propiedad de la herramienta (rechazada — «espera a dos herramientas reales y dos consumidores reales, y luego una unión etiquetada de intención de renderizado»). Esa barra ya está cumplida; esta es esa unión.
- Ampliado por [Diffs de hunk aplicado en tiempo de resultado](../../archived/architecture/2026-07-02-result-time-applied-hunk-diffs.md) (archivado), que añadió un canal `meta` persistido — la separación valor/presentación y el canal persistido `presentationMeta` son ahora propiedad de [el contrato canónico de salida de herramientas](2026-07-20-canonical-tool-output-contract.es.md), de modo que write/edit emiten un `DiffResultView` en tiempo de resultado — el cambio aplicado (un hunk contextual con líneas de contexto / uno por sitio de `replace_all`, o un diff de archivo completo para una creación) — sobre la tarjeta de diff en tiempo de llamada de esta unión.
- Pliega `ToolTerminal` en las vistas `terminal` etiquetadas que usan los transportes de UI actuales.
