# dsh-system-prompt

[English](README.md) | Español

Registro de ensamblaje del system prompt. Los plugins contribuyen secciones ordenadas, schemas de herramientas y variables con nombre. El loop ensambla una vez por paso y renderiza el resultado como el prompt completo del modelo. Este plugin posee la identidad estática del harness y la persona de despliegue global; una persona con ámbito de agent sombrea el valor global por defecto.

## Config

| Clave | Valor por defecto | Significado |
|---|---|---|
| `includeHarnessIdentity` | `true` | Incluye la apertura fija de orden −100 `You are an AI agent powered by DeepSeek Harness.`. Ponlo en `false` solo cuando un despliegue de compatibilidad posee el system prompt completo. |
| `includeRuntimeContext` | `true` | Incluye los contextos dinámicos ordenados en el ensamblaje. Cuando es `false`, los providers de contexto no se evalúan y los contextos añadidos por los listeners de `system-prompt/assemble` se descartan después del waterfall (cascada de eventos); los demás servicios y su aplicación siguen activos. |
| `persona` | `''` | El valor por defecto global de la persona de despliegue: el ÚNICO fragmento de prompt escrito en la configuración, renderizado como la sección `deployment:persona` de orden 0 salvo que una contribución con ámbito de agent lo sombree. Es una plantilla: los grupos `{{…}}` completos se interpretan estrictamente contra las variables registradas (el loop incluido registra `{{model}}`/`{{cwd}}`), y todavía no hay sintaxis de escape para las llaves literales. Vacío ⇒ la sección se elimina al renderizar. |
| `toolOrder` | — | Orden explícito de las herramientas visible para el modelo, como lista de `ToolSchema.name`s con una entrada rest `'<unlisted-tools>'` (`TOOL_ORDER_REST`): las herramientas listadas ocupan su posición listada y las no listadas caen en la entrada rest en orden lexicográfico de nombre. Ausente ⇒ orden lexicográfico de nombre plano. Se aplica a las herramientas recogidas ANTES del waterfall `system-prompt/assemble` — como el ordenamiento por `order` de las secciones, canonicaliza lo que contribuyó el registro (el orden de registro es un artefacto de la carga de plugins), y un listener del waterfall que mute la lista es el dueño del determinismo de lo que emite. La mala configuración falla de forma ruidosa: una lista sin exactamente una entrada rest, o con duplicados, lanza una excepción al cargar; un nombre listado sin herramienta registrada rechaza cada `assemble()`; un provider de herramientas que devuelva el nombre de la entrada rest reservada también rechaza. Bajo el loop incluido, el turno falla antes de cualquier petición al modelo. Por qué una lista central y no pesos por plugin: [Orden explícito de herramientas visible para el modelo](../../../.agents/notes/implemented/feature/2026-07-06-explicit-tool-order.es.md). |

## Servicio: `SystemPrompt` (clave de ctx: `systemPrompt`)

### API pública

- `ctx.systemPrompt.section(section: PromptSection): () => void`: contribuye una sección. La capa es el ámbito del contexto que llama: `agent.ctx` contribuye solo a ese agent, sombreando allí la sección global homónima. Una sección con `complete: true` se convierte, después del waterfall de ensamblaje, en el prompt completo exacto; más de una sección complete efectiva rechaza el ensamblaje. Los nombres duplicados dentro de una capa y los órdenes no finitos lanzan una excepción. Se libera junto con el fiber que realiza la llamada.
- `ctx.systemPrompt.context(context: PromptContext): () => void`: contribuye contexto dinámico ordenado para el ámbito que llama. Los providers se evalúan en cada ensamblaje elegible y se convierten, bajo el loop incluido, en una instantánea del contexto de runtime con su fuente en el historial del modelo.
- `ctx.systemPrompt.suppressRuntimeContext(): () => void`: suprime toda contribución de contexto dinámico para el ámbito que llama. Varios registros se componen de forma independiente; liberar el effect devuelto restaura el contexto cuando no queda ningún supresor.
- `ctx.systemPrompt.tools(provider: (context: AssembleContext) => ToolProviderResult): () => void`: contribuye schemas de herramientas, evaluados en cada ensamblaje con el contexto de ese ensamblaje. `ToolProviderResult` = `{ schemas, knownNames? }`: `schemas` es el conjunto visible posterior a la restricción; `knownNames` es el universo anterior a la restricción que usa `toolOrder`. Un provider no debe devolver un schema llamado `TOOL_ORDER_REST`. Los providers con ámbito solo se consultan para los ensamblajes de su ámbito. Se libera junto con el fiber que realiza la llamada.
- `ctx.systemPrompt.variable(name: string, provider: (context) => string | undefined): () => void`: contribuye una variable de prompt, referenciada desde el texto de la sección como `{{name}}`. Las variables con ámbito sombrean una global homónima para ese agent. Los nombres duplicados en la capa o no referenciables lanzan una excepción; `undefined` significa «sin valor para este ensamblaje». Se libera junto con el fiber que realiza la llamada.
- `ctx.systemPrompt.assemble(context?: AssembleContext): Promise<PromptAssembly>`: ensambla el prompt para un llamante: la capa global fusionada con la capa de `context.scope`, con los schemas de herramientas separados antes del waterfall de transformación. Recorre el waterfall `system-prompt/assemble` filtrado por ámbito y, después, restaura una sección complete efectiva como única sección del prompt y aplica cualquier supresor de contexto de runtime activo. Un `context.signal` opcional controla explícitamente esta petición de ensamblaje; los providers y los listeners pueden cooperar con él, pero no deben conservarlo para otro turno. Rechaza cuando hay varias secciones complete, cuando un `toolOrder` configurado nombra una herramienta fuera del universo `knownNames` de los providers, o cuando un provider devuelve el nombre de la entrada rest reservada.

<a id="live-events"></a>

### Eventos en vivo

`system-prompt/assemble` es la autoridad para las secciones ordinarias; una sección complete es la restricción final del prompt que se aplica después del waterfall. Los listeners que reemplazan entradas deben conservar cualquier protocolo de Code Mode o de salida estructurada activo. Usa [`ToolRuntime.restrict()`](../tools/README.es.md) cuando el filtrado deba mantenerse alineado entre la presentación, la búsqueda y la ejecución. Las notificaciones de cambio del registro no están filtradas. La región generada de [system-prompt.md](../../../docs/subsystems/system-prompt.es.md#cordis-surface) posee las firmas y los contratos de despacho.

### Tipos clave

- `AssembleContext`: el para qué de una llamada a `assemble()`. Extensible mediante fusión; aquí declara `scope?: ScopeKey` (el selector de capa) y `signal?: AbortSignal` (la capacidad de control explícito de la petición), mientras que `dsh-agent` declara `agent?: Agent` (el campo DX tipado: nunca se establece sin `scope`; usa `assembleContextFor(agent, signal)`). Los providers deben tolerar los campos ausentes, porque un `assemble()` desnudo lleva un contexto vacío, sin ámbito y sin señal. `signal` es un valor de la petición, no parte del frame de ejecución ambiental del Agent.
- `PromptSection`: `{ name, order, text, complete? }`. Las secciones se concatenan en `order` ascendente. Bandas de orden: `-100` es la identidad del harness, `0` la persona de despliegue y la guía de herramientas usa `100–199`. Una sección `complete` efectiva suprime todas las demás secciones después del ensamblaje cooperativo.
- `PromptAssembly`: `{ sections: AssembledSection[], tools: ToolSchema[], variables: Record<string, string | undefined> }`. Los textos de las secciones llegan resueltos pero aún no interpolados; `variables` contiene cada variable registrada resuelta contra el contexto. Los schemas de herramientas forman parte del ensamblaje por diseño: «lo que se le dice al modelo que puede hacer» es una sola cosa coherente, aunque los adaptadores transmitan los schemas como un campo wire independiente.
- `renderPrompt(assembly)`: interpola las referencias `{{variable}}` en cada sección, elimina las secciones vacías y une con líneas en blanco. ESTRICTO: una referencia desconocida (búsqueda con `Object.hasOwn`: los nombres de prototipo como `{{constructor}}` son desconocidos), una referencia registrada pero sin valor, un grupo `{{…}}` completo mal formado, o un `{{` que no abre ningún grupo completo mientras todavía lo sigue un `}}` (`{{{model}}}`) lanzan una excepción: fallar de forma ruidosa es mejor que entregar un prompt mal formado. Un `{{` solitario sin ningún `}}` después pasa tal cual; los valores sustituidos nunca se vuelven a escanear.

Extensible mediante fusión: los plugins pueden declarar campos adicionales en `PromptAssembly` y `AssembleContext` mediante fusión de declaraciones.

### Puntos de extensión

- Providers de secciones: los paquetes de herramientas poseen su guía entre llamadas (`tool:bash`, `tool:read`, …); este plugin posee `harness:identity` y `deployment:persona`.
- Providers de variables: el agent loop registra `model` y `cwd`; cualquier plugin puede registrar los hechos que posee (un futuro `date`, el estado de git, …).
- Providers de schemas de herramientas: `ToolRuntime` se registra automáticamente como provider de herramientas.
- El [waterfall `system-prompt/assemble`](#live-events): mutar o reemplazar cooperativamente el ensamblaje por llamante antes de que se aplique cualquier restricción de sección complete.

Razón de diseño: [el Agent Note de variables de prompt](../../../.agents/notes/implemented/architecture/2026-07-05-prompt-variables-and-tool-guidance-ownership.es.md).

## Experiencia del modelo

### System prompt

#### Lo que ve el modelo

Por defecto, cada ensamblaje empieza con la identidad del harness que sigue y, después, la persona configurada y las secciones ordenadas de los plugins tras una interpolación estricta de variables. `includeHarnessIdentity: false` omite solo esa apertura fija. Las secciones vacías desaparecen; las secciones y las variables con ámbito pueden sombrear las globales para un agent. El waterfall `system-prompt/assemble` determina el prompt y los schemas de herramientas entregados salvo que una sección efectiva se declare complete; esa sección exacta se convierte entonces en todo el system prompt, mientras los contextos, las herramientas y las variables del waterfall permanecen. Los contextos dinámicos ordenados son independientes de las secciones del system prompt y solo se convierten en instantáneas con su fuente del rol de usuario cuando están presentes. `includeRuntimeContext: false` o un supresor con ámbito elimina todos esos contextos, incluidos los añadidos por los listeners, sin desactivar los servicios que poseen la política o el estado subyacentes.

##### Identidad del harness

```markdown
You are an AI agent powered by DeepSeek Harness.
```

#### Efecto en tokens

La identidad es un coste fijo por petición cuando está activada. La persona y el texto de los plugins se repiten en cada petición y escalan con su contenido renderizado.

#### Efecto en la caché KV

Estable por prefijo mientras la identidad, la persona, las variables, el texto de las secciones y el orden se rendericen de forma idéntica. Cualquier cambio puede invalidar la reutilización desde el primer token de system prompt cambiado.

### Schemas de herramientas

#### Lo que ve el modelo

Para las herramientas incluidas, el modelo recibe el subconjunto visible por agent de los [schemas de herramientas generados](../../../docs/tool-catalog.es.md#tool-package-map), ordenados por configuración o lexicográficamente después de las restricciones y la interceptación del ensamblaje. Las extensiones pueden contribuir definiciones adicionales a través del mismo registro. Las secciones y los providers de schemas son entradas de ensamblaje independientes, de modo que una restricción de herramienta no elimina la guía registrada por separado.

#### Efecto en tokens

Los tokens de schema se repiten en cada petición. Restringir una herramienta elimina todo su coste de schema para ese agent, pero no una sección de prompt independiente; reordenar cambia la forma de la caché, pero no el contenido semántico.

#### Efecto en la caché KV

Estable por prefijo mientras el conjunto de schemas visible, el renderizado y el orden no cambien. El registro, la restricción o el reordenamiento pueden invalidar la reutilización desde el primer token de schema cambiado.

## Limitaciones conocidas y trabajo diferido

- **El texto de prompt escrito por el despliegue es solo configuración/composición**: este plugin posee la persona global por defecto, los plugins creadores pueden registrar sombreados con ámbito de agent, y las demás secciones vienen del plugin que posee el hecho; no hay API de edición de prompt para el usuario final.
- **No hay sintaxis de escape para las llaves literales `{{…}}`**: cada grupo completo se interpola contra las variables registradas; el escape queda diferido hasta que un prompt real lo necesite.
- **La mala configuración de `toolOrder` aflora en el ensamblaje del prompt (el primer turno), no en el arranque**: solo las violaciones de forma lanzan una excepción al cargar la configuración.
- **Las secciones que comparten un valor de `order` se desempatan por orden de registro**: un artefacto de la carga de plugins; el determinismo se apoya en la convención de bandas de orden distintas, a diferencia del orden de herramientas canonicalizado.
