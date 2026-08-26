# dsh-tools

[English](README.md) | Español

Registro de herramientas y pipeline de ejecución. Los plugins de herramientas registran sus schemas y ejecutores; el agent loop ejecuta cada llamada a través de `tools/pre-execute` (la puerta extensible de allow/deny) → guards monótonos registrados → `tools/execute` (un envoltorio alrededor del despacho para los plugins de tiempo de espera, reintento o métricas) → `tools/post-execute` (inspeccionar/sustituir el resultado, adjuntar contexto) → el límite `finalizeContent` propiedad de la definición → la notificación `tools/result`, solo de observación. El registro también posee el CÓMO se presentan sus herramientas al modelo: su config `mode` selecciona el Function Calling nativo, [Code Mode](#code-mode) o ambos, y un agent sombrea ese valor por defecto para sí mismo con `presentAs`.

## Servicio: `ToolRuntime` (clave de ctx: `tools`)

### Config

```yaml
tools:
  mode: native   # native (default) | code | both
```

`native` contribuye las herramientas visibles como definiciones de función. `code` contribuye el transporte reservado `run_code`, la sección generada `tools:sdk` y la regla `tools:code-only` que establece que solo se puede llamar directamente a `run_code` — que el ejecutor aplica después, resolviendo a `UNKNOWN_TOOL` cualquier llamada directa del modelo que nombre a otra herramienta antes de que la política se ejecute; `both` contribuye ambas formas y no establece tal regla, porque sus llamadas nativas sí se ejecutan. Este es el valor por defecto para los agents que no declaran ninguno propio: un preset de agent elige el suyo con [`dsh-agent-tool-presentation`](../agent-tool-presentation/README.es.md). El transporte reservado no puede registrarse, sombrearse, restringirse ni eliminarse, y su nombre está reservado sea cual sea el modo configurado, porque cualquier agent puede elegir un modo code. Los modos no nativos requieren un `ctx.codeRuntime` cuyo `language` tenga un renderizador de SDK registrado: TypeScript se entrega a través de [`dsh-code-runtime-worker-thread`](../../code-runtime/code-runtime-worker-thread/README.es.md); un renderizador de Python viene integrado y maneja cualquier runtime que informe de `language: 'python'` (un backend de primera parte `dsh-code-runtime-python` se entrega por separado). Un lenguaje de runtime sin renderizador hace fallar el ensamblaje del prompt de forma ruidosa, y una entrada de `systemPrompt.toolOrder` para una herramienta que el modo no contribuye rechaza el ensamblaje del prompt. Un listener de `system-prompt/assemble` puede reemplazar las contribuciones del registro; el ensamblaje que devuelve es autoritativo, de modo que ese listener es el responsable de conservar un protocolo de Code Mode utilizable.

### API pública

- `ctx.tools.register(definition: ToolDefinition): () => void`: registra una definición tipada de confianza del mismo proceso con una declaración `output` canónica obligatoria. La capa es el ámbito del contexto que llama: un contexto de plugin plano registra globalmente; el `agent.ctx` de un agent registra solo para ese agent, sombreando allí una herramienta global homónima. Los nombres duplicados dentro de una capa lanzan una excepción; los modos no nativos también rechazan el nombre del transporte reservado `run_code`. Las declaraciones de salida ausentes o no compatibles y un `timeoutMs` no positivo o no finito fallan en el registro. El callback síncrono opcional `finalizeContent` se captura en instantánea cuando empieza una llamada y solo puede reemplazar el contenido final visible para el modelo después de que se normalice cada resultado del pipeline, incluido un error descubierto al materializar otro campo del resultado. Se libera junto con el fiber que realiza la llamada.
- `ctx.tools.presentAs(mode: ToolPresentationMode): () => void`: selecciona la presentación de este agent visible para el modelo, sombreando el config `mode` solo para ese agent; lanza una excepción desde un contexto plano (la presentación de todo el proceso es el campo de config) y desde una segunda declaración en el mismo ámbito. Un modo code también registra la sección `tools:sdk` propia de ese agent. El catálogo no cambia: `schemas(agent)` sigue informando de las capacidades del agent; solo colapsan las herramientas del ensamblaje. Se libera junto con el fiber que realiza la llamada.
- `ctx.tools.restrict(filter)`: aplica una máscara de allow/deny con ámbito de agent a las herramientas globales y lanza una excepción desde un contexto plano. El filtro se captura en instantánea en el registro; varias máscaras se intersectan y las herramientas locales al ámbito se fusionan después. Las máscaras de deny admiten globales posteriores sin nombrar, mientras que las máscaras de allow excluyen nombres posteriores. Los nombres desconocidos, locales o reservados y los filtros vacíos se rechazan. Es composición de visibilidad en vivo, no un límite de autoridad; consulta el [objetivo no de seguridad del ámbito](../../../.agents/notes/implemented/architecture/2026-07-08-agent-scope-contexts.es.md#security-and-authority-are-non-goals).
- `ctx.tools.get(name: string, scope?: ScopeKey): ToolDefinition | undefined`: la resolución tal como la ve un ámbito (con el sombreado aplicado; un global restringido se lee como ausente): los presentadores pasan el agent que llama para que la tarjeta coincida con lo que se ejecutó.
- `ctx.tools.schemas(scope?: ScopeKey): ToolSchema[]`: los schemas de todo lo que el ámbito puede ver (sin las funciones `execute`). Los schemas de las herramientas incluidas están catalogados en [docs/tool-catalog.md](../../../docs/tool-catalog.es.md), generados arrancando cada plugin de herramientas y recogiendo este método (consulta el [Agent Note del catálogo de schemas de herramientas](../../../.agents/notes/implemented/process/2026-07-02-tool-schema-catalog.es.md)).
- `ctx.tools.guard(guard: ToolGuard): () => void`: registra un guard de ejecución síncrono monótono después de `tools/pre-execute`: devolver una razón deniega la llamada, mientras que `undefined` la deja sin cambios. Un guard de contexto plano se aplica globalmente; un guard de `agent.ctx` se aplica solo a ese agent. Los listeners posteriores del waterfall no pueden convertir una denegación de guard en un permiso. Se libera junto con el fiber que realiza la llamada.
- `ctx.tools.execute(exec)`: captura y congela los argumentos sin pérdida, asigna un token opaco, ejecuta el pipeline completo de política/despacho/resultado y, después, captura en instantánea de forma independiente el resultado autoritativo antes de la observación final. Los argumentos no válidos usan la misma vía de resultado sin llegar a la política ni al cuerpo. Los envoltorios around solo pueden reemplazar `signal`; el registro vuelve a fusionar la señal original del llamante inmediatamente antes del cuerpo.
- `ctx.tools.executionMode(exec)`: devuelve `parallel` solo cuando el clasificador `isConcurrencySafe(exec.arguments)` de la definición visible devuelve exactamente `true`; las clasificaciones desconocidas, ocultas, no declaradas, no válidas o que lanzan excepciones son exclusivas.

### Servicios inyectados

`SystemPrompt`: el registro introduce automáticamente sus schemas de herramientas en el ensamblaje del system prompt a través de `ctx.systemPrompt.tools()`. En cambio, el seam de aprobación se consume de forma oportunista (`ctx.get('approval')`, sin inyección estática): un despliegue sin él conserva la degradación ask→deny, y el registro permanece activo en cualquier caso.

### Cancelación

La cancelación es cooperativa y en calma. Cada invocación tipada aporta un `AbortSignal` propiedad del llamante; los cuerpos de las herramientas lo reciben como el `exec.signal` de solo lectura obligatorio, mientras que solo los envoltorios de `tools/execute` pueden reemplazar temporalmente la señal obligatoria. El registro conserva la cancelación del llamante a través del reemplazo y nunca abandona una promise del mismo proceso ya iniciada. La cancelación anterior a la invocación del cuerpo es `ABORTED_BEFORE_DISPATCH`; la cancelación posterior a la invocación solo puede reemplazar un resultado de éxito por `ABORTED`. Una denegación, un fallo de envoltorio, un fallo de herramienta, un fallo posterior a la política o un `TOOL_TIMEOUT` propiedad del tiempo de espera siguen siendo más específicos. Una entrada ya abortada materializa y congela los argumentos, omite todas las fases de política y despacho y publica un único resultado. Cada herramienta asíncrona debe observar o reenviar la señal y concluir solo después de que se detenga el trabajo que le pertenece. El [Agent Note de cancelación de herramientas](../../../.agents/notes/implemented/architecture/2026-07-19-cooperative-tool-cancellation.es.md) posee el contrato completo y el límite de terminación dura.

### Eventos en vivo

El pipeline vivo del registro tiene tres waterfalls transformables, después el finalizador de contenido propiedad de la definición y, por último, el evento `tools/result` solo de observación; los cambios del registro son notificaciones deliberadamente sin filtrar de estado compartido. Las firmas exactas, los modos de despacho, el filtrado por ámbito y los contratos de contención de fallos viven en la región generada de [tools.md](../../../docs/subsystems/tools.es.md#cordis-surface), mientras que el orden completo se visualiza en la [pipeline de ejecución de herramientas generada](../../../docs/tool-execution-pipeline.es.md). `tools/result` es en vivo; el `tool/result`, de nombre similar, es el evento de sesión duradero que el agent loop añade después.

### Tipos clave

- `ToolDefinition`: `ToolSchema` + `output { schema, render, presentationMeta? }` obligatorio + `execute(args, exec)`, callbacks opcionales de contenido final y presentación, `timeoutMs` cooperativo y clasificación opcional por llamada `isConcurrencySafe(args)`. Un cuerpo devuelve solo el valor JSON canónico declarado por el schema de salida y se detiene de forma cooperativa a través de `exec.signal`. `finalizeContent(exec, result)` se ejecuta exactamente una vez para cada resultado normalizado, incluidos los fallos que omiten la fase posterior a la política, y solo puede reemplazar `content`; debe ser síncrono y total.
- `ToolExecutionInput`: la descripción de la llamada que aporta el llamante: `{ callId, name, arguments, signal, agent?, parent? }`; `signal` es obligatorio y de solo lectura, los llamantes pueden pasar el token opaco de una ejecución envolvente como `parent`, y los llamantes nunca eligen el token propio de la nueva ejecución.
- `ToolExecutionToken`: un `Symbol` con marca recién creado que asigna el registro. Solo admite correlación por igualdad y nunca cruza un límite de modelo, log o worker.
- `ToolExecution`: la vista de solo lectura del pipeline: el inmutable `{ token, callId, name, arguments, signal, agent?, parent? }`; el registro conserva por separado y vuelve a fusionar la señal original del llamante. `ToolDispatchExecution` es la vista exclusiva de `tools/execute` cuya señal obligatoria es mutable, de modo que un envoltorio puede sustituirla y restaurarla, pero no eliminarla. El `parent` de una llamada anidada es un `ToolExecutionToken`, no un objeto de ejecución.
- `ToolRunContext`: la ejecución que se pasa al cuerpo de una herramienta, que extiende `ToolExecution` con `deferContext(context)`. Difiere un contexto hasta que el resultado final de la herramienta llega al loop — normalmente un contexto de despacho anidado transportado por una herramienta compuesta, o una instrucción nueva con origen en un plugin acuñada por una herramienta hoja (el wrap-up de `tool-goal`) — incluso cuando la herramienta lanza después o gana la cancelación; nunca inyecta de inmediato.
- `ToolExecutionResult`: desenlace discriminado, local a la ejecución. El éxito es `{ isError:false, value:JsonValue, content, meta?, additionalContexts? }`; el fallo es `{ isError:true, error:{ message, info? }, content, meta?, additionalContexts? }` y no tiene valor. La identidad de la llamada permanece en el `ToolExecution` inmutable. El registro toma una instantánea, valida y congela el valor canónico antes de renderizar y, después, materializa los campos de presentación duraderos antes de la observación final. `ToolFailure.info` lleva un `{ name, code }` interno para un `HarnessError`; `additionalContexts` conserva cada `UserMessage` diferido o identificado en post-execute para la FIFO posterior al resultado del loop.
- `PreToolDecision`: `{kind:'allow'}` | `{kind:'deny', reason}` | `{kind:'ask', reason?}`. La reescritura de entrada no se ofrece a propósito; `ask` lo atiende [`ctx.approval`](../../interaction/user-approval/README.es.md) cuando está montado y, de lo contrario, degrada a deny.
- `PostToolDecision`: accept puede reemplazar `content` o `value`, nunca ambos, y puede adjuntar `additionalContexts`; block convierte la retroalimentación en un fallo sin valor. El reemplazo de contenido conserva el valor canónico y los metadatos. El reemplazo de valor se revalida y vuelve a renderizar content/metadata. accept conserva los contextos diferidos por la herramienta antes que los contextos de la decisión; block descarta los contextos diferidos por la herramienta y expone solo los contextos que aporta explícitamente la decisión de bloqueo.
- `ToolGuard`: `(execution) => string | undefined`; la cadena devuelta es una razón de denegación monótona final, evaluada después del waterfall reordenable de pre-execute y antes del despacho.
- `ToolCallView` / `ToolResultView`: intenciones de renderizado neutrales respecto al provider y etiquetadas como `card` que una herramienta devuelve desde `presentCall` / `presentResult` para ser la dueña de cómo una UI renderiza SUS llamadas (consulta «Presentación de UI propiedad de la herramienta»).

### Puntos de extensión

- Los plugins de herramientas llaman a `ctx.tools.register()`: los schemas fluyen al ensamblaje automáticamente.
- `tools/pre-execute` es la puerta reordenable de allow/deny/ask; `ctx.tools.guard()` añade después política de propietario monótona.
- `tools/execute` envuelve el despacho canónico normalizado para el tiempo de espera, el reintento o las métricas. Los envoltorios solo pueden reemplazar la señal operativa; un éxito creado por un envoltorio se normaliza a través de la declaración de salida de la herramienta resuelta. Cada resultado canónico pertenece a un token de despacho inmutable, de modo que un resultado en caché de otra llamada u otra herramienta se revalida bajo la declaración activa.
- `tools/post-execute` puede reemplazar el contenido de presentación, reemplazar el valor canónico, bloquear con retroalimentación o adjuntar contextos ordenados. El `finalizeContent` opcional de una definición es entonces el dueño de su último invariante solo de contenido, tanto en los resultados normales como en los fallos externos del pipeline; `tools/result` observa el desenlace final inmutable. El reemplazo de contenido no es un límite de confidencialidad: bloquea o reemplaza el valor cuando los consumidores programáticos no deben recibirlo.
- Las firmas exactas y el orden viven en la región generada de [tools.md](../../../docs/subsystems/tools.es.md#cordis-surface) y de la [pipeline](../../../docs/tool-execution-pipeline.es.md).
- Servidores MCP: un plugin por servidor, descubre las herramientas y llama a `ctx.tools.register()` con los schemas del servidor.

### Schemas de parámetros de herramientas tipados

Los autores de plugins de primera parte pueden usar el helper `defineTool()` (exportado desde este paquete) para schemas de parámetros de herramientas tipados:

```ts
import { readFile } from 'node:fs/promises'
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

declare const ctx: Context

ctx.tools.register(defineTool({
  name: 'read_file',
  description: 'Read a file from disk.',
  parameters: {
    path: { type: 'string', required: true, description: 'Absolute file path' },
    offset: { type: 'number' },
    limit: { type: 'number' },
  },
  output: {
    schema: { type: 'string' },
    render: (_args, value) => [{ type: 'text', text: value }],
  },
  async execute(args, exec) {
    // args is typed: { path: string; offset?: number; limit?: number }
    return readFile(args.path, { encoding: 'utf8', signal: exec.signal })
  },
}))
```

El DSL de schema unificado usa `ParameterSchemaSpec` para el objeto de parámetros abierto implícito y `ValueSchemaSpec` para cualquier raíz de valor JSON. Admite `string`, `number`, `integer`, `boolean`, `null`, `array`, `object`, `json` solo para autores y `oneOf` de exactamente uno; los valores escalares de `enum`/`const` son correctos a nivel de tipos. Cada objeto DSL explícito declara `additionalProperties: true | false`, mientras que la raíz de parámetros implícita y el JSON Schema crudo conservan el valor por defecto abierto estándar. Los registros de schema solo aceptan claves de cadena propias y enumerables, y los arrays de schema deben ser arrays ordinarios densos. La compilación, la validación, el desacoplamiento del registro y el renderizado de schema a TypeScript usan pilas de trabajo explícitas, de modo que el procesamiento en runtime de schemas profundos válidos está acotado por memoria y no por pila de llamadas; `InferValue` conserva los tipos exactos a través de 16 niveles de contenedor y después recurre a `JsonValue` para que el propio TypeScript siga siendo seguro para la pila.

Una definición de `defineTool` valida los argumentos del modelo antes de la ejecución y convierte los valores obligatorios ausentes, los primitivos incorrectos, los miembros de enum no válidos y las violaciones anidadas en `ToolArgsError` (`INVALID_ARGS`) para la vía normal de resultado de error. También infiere el retorno del cuerpo y los proyectores de salida puros a partir de `output.schema`; el registro toma una instantánea y valida el JSON sin pérdida devuelto antes de la presentación. La raíz de parámetros implícita está abierta; un objeto explícito acepta claves adicionales solo con `additionalProperties: true`, y un objeto cerrado sin propiedades declaradas acepta solo `{}`. Los objetos JSON Schema crudos permanecen abiertos salvo que establezcan explícitamente `additionalProperties: false`. Los valores por defecto no se aplican; los objetos abiertos sin `properties` y los arrays sin `items` solo reciben una comprobación de tipo contenedor. Las herramientas registradas en crudo poseen la validación de entrada, pero aun así declaran y reciben una salida aplicada por el registro.

Consulta `defineTool`, `validateArgs`, `ToolArgsError`, `ValueSchemaSpec`, `ParameterSchemaSpec`, `InferValue`, `InferArgs`, `valueSchemaSpecToJsonSchema` y `parameterSchemaSpecToJsonSchema` en la API pública para más detalles.

El `timeoutMs` opcional debe ser positivo y finito; es metadatos de política, no schema visible para el modelo.

El `isConcurrencySafe(args)` opcional recibe argumentos tipados y validados suavemente. Solo el `true` exacto permite la ejecución concurrente de despacho/cuerpo; la entrada no válida y todos los demás resultados siguen siendo exclusivos. Los cuerpos que se acogen a la concurrencia no mutan el estado propiedad del padre, y las carreras de estado compartido deben conmutar o fallar en cerrado. El [Agent Note de llamadas de herramienta paralelas](../../../.agents/notes/implemented/feature/2026-07-10-parallel-tool-call-execution.es.md) posee el contrato de seguridad completo.

### Subconjunto JSON Schema crudo aplicado

`JsonSchemaNode` es la contraparte cruda que comparten las salidas de herramientas, la generación de Code Mode, los subagentes y los flujos de trabajo. Permite cualquier raíz JSON, un nodo JSON sin restricciones solo de anotaciones y `oneOf` de exactamente uno; las anotaciones deben seguir siendo JSON sin pérdida. `assertSupportedJsonSchema()` rechaza los constructos no compatibles, mientras que `validateJsonSchemaValue()` devuelve violaciones calificadas por ruta. Los subagentes y los flujos de trabajo conservan su requisito de raíz de objeto definido por el llamante a través de `assertObjectJsonSchema()` y `ObjectJsonSchema`, no mediante una limitación del vocabulario compartido.

### Presentación de UI propiedad de la herramienta

Las herramientas poseen opcionalmente intenciones de renderizado puras `presentCall()` y `presentResult()`, de modo que las UIs no tratan los nombres de herramientas como casos especiales:

- Las vistas de llamada son `{ card: 'generic', title, kind?, rawInput?, content?, locations? }`, `{ card: 'terminal', title, description?, cwd? }` o `{ card: 'diff', title, diffs, locations? }`.
- Las vistas de resultado son `{ card: 'generic', title?, content? }`, `{ card: 'terminal', title?, output?, exitCode?, signal? }`, `{ card: 'diff', title?, diffs }`, `{ card: 'search', shape, title?, truncated, total, … }` (una búsqueda de descubrimiento completada: coincidencias agrupadas por archivo para `shape: 'matches'` (grep) o una lista plana de rutas para `shape: 'paths'` (glob), con `truncated`/`total` para que una UI nunca presente un resultado recortado como completo; la vista no lleva texto de resultado y una búsqueda no tiene un análogo de tiempo de llamada `card: 'search'`), `{ card: 'read', title?, path, offset, lines, totalLines, lang?, content? }` (una lectura de archivo completada: una vista de código numerada por líneas y opcionalmente resaltada con sintaxis; `offset` es la primera línea con base 1 que pidió la ventana y se conserva incluso cuando `lines` está vacío; `lines` es `{ number, text }[]` y conserva el número de cada línea del archivo, y `content` es el texto sin envoltorio al que recurre una UI sin soporte de lectura), o `{ card: 'web', kind: 'search' | 'fetch', title?, … }` (una recuperación web completada; los brazos de `kind` llevan las fuentes de búsqueda estructuradas o el resumen de fetch, y una UI sin la capacidad `web` recurre al contenido crudo del resultado).

Devolver `undefined` selecciona la alternativa genérica. Los presentadores dependen solo de sus argumentos y del resultado duradero, porque las UIs los llaman durante la transmisión en vivo y la reproducción de logs. `output.presentationMeta(args, value)` deriva metadatos JSON para las llamadas directas de nivel superior; esos metadatos persisten con `tool/result` y vuelven a `presentResult`, mientras que el valor canónico en sí permanece local a la ejecución y nunca se reproduce. Los despachos Code anidados no calculan metadatos. `defineTool` valida suavemente los argumentos registrados antiguos y recurre a la alternativa en lugar de romper la reproducción. `dsh-tool-bash` y `dsh-tool-fs` son las implementaciones de referencia; el [Agent Note de salida canónica](../../../.agents/notes/implemented/architecture/2026-07-20-canonical-tool-output-contract.es.md) posee la división valor/presentación, y el [Agent Note de intención de renderizado](../../../.agents/notes/implemented/architecture/2026-07-02-tool-render-intent-union.es.md) posee el vocabulario de tarjetas.

### Code Mode

Bajo `code` o `both`, el registro expone el transporte reservado `run_code` y un SDK determinista para el ámbito actual, generado en el lenguaje del runtime cargado: el registro selecciona el renderizador por `ctx.codeRuntime.language` (`typescript` → el SDK de TypeScript que sigue, `python` → el SDK de Python). El SDK declara tipos exactos de argumentos por herramienta y de salida canónica para cada herramienta visible (`ToolArgsMap`/`ToolOutputMap` en TypeScript, `TypedDict`s con nombre en Python), y cada binding se resuelve al valor JSON canónico de la herramienta. Cada llamada de binding JSON sin pérdida vuelve a entrar en el pipeline completo de herramientas bajo el contrato de programación nativo (las llamadas seguras para concurrencia pueden solaparse hasta `maxParallelSubCalls`; las llamadas exclusivas se ejecutan solas como barreras de orden) con correlación registrada con la llamada externa. Las denegaciones y otros resultados fallidos se rechazan con el `ToolCallError` real visible para el programa, que solo lleva `toolName` y `message`; el contenido nativo y los códigos de error internos quedan fuera del contrato Code. Los logs externos del programa y su valor de retorno vuelven a entrar en el contexto del modelo; cuando el contenido Native final de un sub-despacho concluido con éxito contiene una imagen, el bridge también difiere ese contenido ordenado completo a través del resultado del padre para que la imagen no se pierda detrás del binding solo JSON. El bloqueo final en post-execute o el reemplazo de contenido es autoritativo. Los efectos secundarios ordinarios no se deshacen, y los `additionalContexts` de los sub-despachos se difieren a través del resultado del padre para conservar la adyacencia llamada/resultado. La conclusión de la ejecución aborta y vacía los bindings pendientes; los fallos de runtime afloran como `CodeRunFailedError`.

Bajo `code` — no bajo `both` — el transporte es también la única entrada que el modelo puede usar: una llamada directa del modelo que nombre a cualquier otra herramienta visible se resuelve a `UNKNOWN_TOOL` en la creación de la ejecución, antes de `tools/pre-execute`, de la aprobación `ask` y de los guards, de modo que nada observa ni aprueba una llamada que solo puede fallar. La denegación nombra la vía de retorno (`only \\`run_code\\` is callable directly — call \\`<name>\\` from inside a \\`run_code\\` program instead`), porque el mismo prompt declara esa herramienta y un `unknown tool` desnudo se lee como un despliegue roto. Los sub-despachos del SDK llevan el token `parent` de la ejecución externa y están exentos, de modo que los programas conservan todos los bindings que declaró el SDK. Consulta la [nota de colapso del ejecutor](../../../.agents/notes/implemented/bug-fix/2026-08-07-code-mode-executor-collapse.es.md), los [fundamentos de Code Mode](../../../.agents/notes/implemented/feature/2026-06-15-code-mode.es.md), el [contrato de retornos tipados](../../../.agents/notes/implemented/feature/2026-07-20-code-mode-typed-tool-returns.es.md) y el [seam code-runtime](../../code-runtime/README.es.md). Prueba `pnpm run demo:code-mode`.

- **La sección SDK** (`tools:sdk`, orden 150): una sección de prompt perezosa que regenera el texto del SDK adecuado al lenguaje en cada ensamblaje. En la variante TypeScript emite `JsonValue`, los `ToolArgsMap` / `ToolOutputMap` exactos, `ToolName`, la declaración de `ToolCallError` y un namespace `tools` mapeado para las capacidades finales visibles del ámbito que llama (nombres exóticos mediante claves entre comillas), además de instrucciones de uso fijas; la variante Python (`ctx.codeRuntime.language === 'python'`) emite los `TypedDict`s con nombre equivalentes y un objeto `tools` con instrucciones de uso equivalentes. Determinista: orden de herramientas lexicográfico, texto idéntico byte a byte para un conjunto de herramientas sin cambios (amigable con la caché de prefijos). Ambos generadores de código se exportan y nunca lanzan durante el ensamblaje del prompt: `jsonSchemaToTs` maneja cada constructo del schema unificado y degrada los constructos crudos no compatibles a `unknown`; `jsonSchemaToPy` hace lo mismo, degradando a `Any` (y un objeto completo a `dict[str, Any]` cuando un nombre de campo no es un atributo `TypedDict` legal, o siempre que se llame fuera del render del SDK, que aporta el contexto de nombres que una declaración `TypedDict` necesita).
- **El bridge de despacho** (el `execute` de `run_code`): cada llamada de binding se captura como JSON sin pérdida antes del despacho (`undefined`, `BigInt`, ciclos, arrays dispersos, `-0` y objetos exóticos rechazan esa llamada), se programa a través de un pool por ejecución que reutiliza el contrato de concurrencia nativo: las llamadas empiezan estrictamente en orden de envío, las llamadas `isConcurrencySafe` consecutivas se solapan hasta el config validado `maxParallelSubCalls` (por defecto 10; `1` restaura el despacho en serie), y una llamada clasificada como exclusiva vacía el pool, se ejecuta sola y bloquea las llamadas posteriores: recibe el token opaco de la ejecución externa como `parent` y recorre el pipeline completo pre-execute → guards → execute → post-execute → result. Un éxito devuelve el valor canónico final después de la política; un fallo llega al worker como un único mensaje y se convierte en `ToolCallError(toolName, message)`. Cada sub-despacho iniciado registra un evento `tool/code-dispatch-start` (id determinista `<parent>:code:<n>`, numerado por envío) a la entrada del pipeline y concluye con un evento `tool/code-dispatch` que lleva el resultado completo `content`/`isError` visible para el modelo (el vocabulario de `tool/result`, de modo que las UIs renderizan los sub-despachos por la vía nativa; los campos `time` del par llevan la temporización de cada sub-despacho); una llamada en cola abandonada por la conclusión de la ejecución no registra ninguno de los dos. `deriveMessages()` no saca a la luz ninguno de los dos eventos ni persiste el valor canónico. La correlación por token permite a los observadores de estilo commit diferir un éxito interno hasta el resultado final de `run_code` sin exponer la ejecución externa en vivo; los efectos secundarios ordinarios de las herramientas no se deshacen. Cada entrada `additionalContexts` de sub-despacho y cada secuencia de contenido final con éxito que contenga una imagen se difieren a través del `ToolRunContext` externo en orden de despacho; el loop añade esos contextos solo después del resultado del `run_code` padre, conservando la adyacencia y la atribución de fuente incluso cuando el programa falla después.
- **Disciplina de conclusión**: el bridge posee un abort con ámbito de ejecución que sigue la señal externa y se dispara cuando la ejecución concluye por cualquier motivo, de modo que la expiración de un presupuesto aborta una sub-herramienta en vuelo en lugar de dejarla huérfana; el bridge vacía entonces su cola ANTES de devolver, de modo que cada `tool/code-dispatch` aterriza dentro del turno abierto. Una ejecución fallida lanza `CodeRunFailedError` (`code: 'CODE_RUN_FAILED'`, message = el tipo de fallo + los logs capturados), que el pipeline convierte en un `isError` estructurado del que el modelo se autocorrige.
- **Tamaño del resultado**: los valores intermedios de los bindings cruzan el proceso del worker enteros y no tienen tope de bytes por binding. `run_code` devuelve el canónico `{ logs: string[], result?: JsonValue }`; las cadenas se renderizan en crudo, cualquier otra raíz JSON presente se renderiza mediante un recorrido pretty JSON seguro para la pila cuya sangría total está limitada a diez caracteres (los subárboles más profundos se mantienen compactos), `null` permanece explícito y la ausencia de `result` significa que el programa devolvió `undefined`. El `maxOutputBytes` configurable del worker (por defecto 64 MiB) se aplica solo a las cargas útiles combinadas y serializadas del array de logs externo, del valor de finalización o de los mensajes de fallo; la sintaxis fija del envoltorio de resultado y el espacio en blanco de presentación quedan fuera de ese límite. Las finalizaciones no válidas o que superan el límite fallan explícitamente, y solo este resultado externo es elegible para el spill ordinario.

### Ejecución paralela

El agent loop agrupa las llamadas `parallel` consecutivas en un pool rodante acotado y trata cada llamada `exclusive` como una barrera de orden. Solo se solapan despacho/cuerpo; la política, los resultados duraderos y el contexto conservan el orden del modelo. Los bindings de Code Mode reutilizan la misma clasificación a través del pool propio del bridge. El [Agent Note de llamadas de herramienta paralelas](../../../.agents/notes/implemented/feature/2026-07-10-parallel-tool-call-execution.es.md) posee las declaraciones incluidas y su razón de ser.

## Experiencia del modelo

### Schemas de herramientas normales

#### Lo que ve el modelo

En el modo normal, el modelo ve el nombre, la descripción y el JSON schema exactos de cada definición visible; las definiciones incluidas se registran en las [secciones generadas de mapa de paquetes de herramientas y schemas](../../../docs/tool-catalog.es.md#tool-package-map). Las restricciones con ámbito de agent, los sombreados y los registros de extensión cambian el conjunto final de herramientas de ese agent.

#### Efecto en tokens

Coste fijo por petición proporcional a las definiciones visibles. Las restricciones que ocultan herramientas eliminan todo su coste de schema para ese agent.

#### Efecto en la caché KV

Estable por prefijo mientras las definiciones visibles y su orden no cambien. El registro, la liberación o una restricción con ámbito pueden invalidar la reutilización desde el primer token de schema cambiado.

### Schema y system prompt de Code Mode

#### Lo que ve el modelo

Code Mode expone el [`run_code` schema generado](../../../docs/tool-catalog.es.md#deepseek-aidsh-tools), las instrucciones del SDK que siguen y el bloque SDK exacto generado para el lenguaje del runtime cargado (el bloque TypeScript `declare const tools` o la declaración Python `tools`). `both` expone los schemas normales y esta API de Code Mode. Bajo `code`, el prompt lleva también la regla `tools:code-only`, ordenada delante de la banda de guía por herramienta para que el modelo lea qué herramientas puede llamar antes de leer para qué sirve cada una; `both` la renderiza vacía. Las instrucciones y el bloque SDK coinciden con el lenguaje del runtime cargado; aquí abajo se muestra la versión TypeScript (a través de [`dsh-code-runtime-worker-thread`](../../code-runtime/code-runtime-worker-thread/README.es.md)), y la versión Python (para cualquier runtime que informe de `language: 'python'`) tiene las mismas operaciones y tipos en sintaxis Python (`await tools.name(args)`, acceso por índice para nombres exóticos, `print(...)` y `return` de nivel superior).

##### Instrucciones del SDK de Code Mode

```markdown
## Writing code for run_code

`run_code` takes two required arguments: `code` — the body of an async TypeScript function (erasable syntax only — no `enum` or namespaces; type annotations are advisory, the code runs type-stripped) — and `description`, a short summary of what the program does. Inside the program:

- Call tools as `await tools.name(args)` — quoted access for exotic names: `tools["my-tool"](args)`. Every call resolves to the tool's typed canonical JSON value. Tool arguments must be lossless JSON.
- A FAILED tool call rejects with `ToolCallError`, whose `toolName` identifies the failed tool and whose `message` is human-readable — `try/catch` it to handle and continue.
- Independent read-only calls MAY overlap under `Promise.all` (safe calls run concurrently; mutating calls run alone, in submission order). Sequence dependent work with `await`.
- Emit results with `return` and/or `console.log(...)`. ONLY what you print or return comes back to you — intermediate tool results never enter the conversation, so extract just what you need.

The available tools:
```

#### Efecto en tokens

Coste fijo por petición proporcional a las definiciones visibles. Code Mode intercambia los schemas de las herramientas finales por texto de SDK generado más un schema de transporte, en lugar de prometer una reducción universal.

#### Efecto en la caché KV

Estable por prefijo mientras la selección de Code Mode, el SDK generado, el schema de transporte y el conjunto de herramientas visibles no cambien. Los cambios de modo o de filtro pueden invalidar la reutilización desde el primer token de prompt o schema cambiado.

### Historial y resultados de llamadas de herramienta

#### Lo que ve el modelo

El loop conserva los argumentos emitidos por el modelo y el contenido final del registro. Cualquier llamada que lance una excepción o sea denegada se convierte exactamente en `Error: <message>`. Code Mode renderiza las líneas impresas del programa externo y su valor de retorno, `(run_code completed with no output)` cuando ambos están vacíos, o `Error: code run failed (<kind>): <message>` seguido condicionalmente de `Captured output:` y de las líneas capturadas. Los eventos de despacho internos permanecen solo en el log, mientras que un sub-resultado con éxito que contenga una imagen se añade después del resultado externo como contexto con atribución de fuente; los listeners de post-execute pueden añadir otro contexto con atribución de fuente en el mismo límite.

#### Efecto en tokens

Los argumentos, los resultados y el contexto adicional dependen de los datos y se reenvían hasta la compactación. Las restricciones que ocultan herramientas también eliminan sus schemas antes de que el modelo pueda llamarlas.

#### Efecto en la caché KV

Solo de anexado (append-only); el contenido recién visible sigue el prefijo de petición reutilizable y no invalida las entradas existentes de la caché KV.

## Limitaciones conocidas y trabajo diferido

- **La política de concurrencia no es una puerta de eventos**: `executionMode()` lee directamente la definición de herramienta resuelta; los plugins solo pueden declarar un clasificador en las definiciones que les pertenecen.
- **`tools/pre-execute` no puede reescribir `exec.arguments` deliberadamente**: los argumentos registrados y renderizados se desincronizarían de lo que se ejecutó; el diseño de la reescritura está en [un Agent Note propuesto](../../../.agents/notes/proposed/feature/2026-06-30-pre-tool-input-rewrite.es.md).
- **Las salidas estructuradas de subagent y flujo de trabajo definidas por el llamante siguen con raíz de objeto**: es una guardia a nivel de Consumer; el vocabulario de schema compartido y las salidas de herramientas admiten cualquier raíz JSON.
- **El `timeoutMs` de una definición es solo declarativo**: el registro nunca aplica plazos; la aplicación requiere el envoltorio `@deepseek-ai/dsh-tool-call-timeout-policy`.
- **El lenguaje del SDK de Code Mode sigue al único runtime cargado, y la presentación es por agent y no por herramienta**: `mode: code`/`both` rechaza el ensamblaje del prompt salvo que `ctx.codeRuntime.language` tenga un renderizador de SDK registrado (TypeScript o Python); las restricciones/sombreados con ámbito y `presentAs` eligen los bindings visibles de cada agent y su forma, pero dentro de un mismo agent ninguna herramienta puede ser solo nativa mientras otra es solo code.
- **Los valores intermedios de Code Mode son locales a la ejecución y sin límite de bytes**: los valores tipados canónicos no pueden reconstruirse desde la reproducción de sesión y pueden agotar la memoria del proceso o del worker; solo la salida externa de `run_code` tiene el tope duro configurable del worker. La copia duradera en el log de cada sub-despacho SÍ está acotada: el waterfall `tools/code-dispatch-log` permite a la política de spill reemplazar un contenido `tool/code-dispatch` sobredimensionado por una vista previa + localizador ([razón](../../../.agents/notes/implemented/feature/2026-07-26-code-dispatch-log-spill.es.md)).
- **El estado de `run_code` es nuevo en cada ejecución**: un kernel persistente estilo REPL se rechaza para el MVP (el estado entre llamadas sería invisible para el log); consulta [el Agent Note de Code Mode](../../../.agents/notes/implemented/feature/2026-06-15-code-mode.es.md).
