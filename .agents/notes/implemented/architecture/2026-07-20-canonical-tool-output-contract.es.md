# Agent Note: Contrato canónico de salida de herramientas

Status: implemented

[English](2026-07-20-canonical-tool-output-contract.md) | Español

## Problema

Los cuerpos de las herramientas escribían antes directamente `ContentBlock[]` orientados al modelo, envolviéndolos opcionalmente con `meta` opaco. La llamada a funciones nativa tenía por tanto una proyección humana utilizable, pero un llamador programático no tenía ningún valor de dominio estable: Code Mode aplanaba los bloques de vuelta a un string, las herramientas dinámicas repetían la forma del contenido, y la política podía reemplazar la presentación sin ninguna forma de distinguir ese cambio de reemplazar el resultado de la operación. Varios seams de capacidad ya devolvían valores de provider más ricos solo para descartarlos en su frontera de herramienta orientada al modelo.

El contrato durable de sesión hacía que esa presentación fuera autoritativa para el replay, pero persistir cada valor intermedio rico ampliaría los registros, expondría datos de implementación a la compactación y la migración, y convertiría incorrectamente una API local de ejecución en formato de sesión. La base necesita en cambio un valor tipado único durante la ejecución y una proyección explícita al contenido durable existente orientado al modelo.

## Decisión

Cada herramienta declara una salida canónica obligatoria y devuelve solo el valor descrito por ella:

```ts ignore-check
output: {
  schema: OutputSchema
  render(args, value): ContentBlock[]
  presentationMeta?(args, value): JsonValue
}
```

`defineTool` infiere el retorno del cuerpo y ambos proyectores a partir del `ValueSchemaSpec` unificado. Las definiciones crudas y dinámicas proporcionan la forma compilada `JsonSchemaNode`. El registro rechaza una declaración ausente o un schema crudo no soportado; no hay ruta de compatibilidad de retorno de contenido.

Para cada despacho correcto el registro toma una instantánea del valor devuelto como `JsonValue` sin pérdida, lo valida contra `output.schema`, lo congela en profundidad y luego invoca el renderizador puro y, para una llamada de superficie directa, el proyector de metadatos opcional. Los fallos de renderizador, proyector, schema o JSON sin pérdida se contienen como resultados ordinarios `ToolOutputError`. Un wrapper alrededor de `tools/execute` recibe y devuelve la unión canónica de éxito/fallo; un éxito escrito por un wrapper se normaliza de nuevo a través de la declaración de salida de la herramienta resuelta en lugar de confiar en contenido escrito de forma independiente. Cada resultado canónico se ata al token de despacho inmutable que lo creó, de modo que devolver un resultado cacheado de otra llamada o herramienta dispara la normalización bajo la declaración activa en lugar de eludirla.

```ts ignore-check
type ToolExecutionResult =
  | { isError: false; value: JsonValue; content: ContentBlock[]; meta?: JsonValue; additionalContexts?: HookContext[] }
  | { isError: true; error: { message: string; info?: { name: string; code: string } }; content: ContentBlock[]; meta?: JsonValue; additionalContexts?: HookContext[] }
```

`tools/post-execute` tiene dos proyecciones de éxito mutuamente excluyentes. Reemplazar `content` cambia solo la presentación Native/Model y conserva el valor canónico y los metadatos. Reemplazar `value` revalida el reemplazo y recalcula ambas proyecciones de presentación. Un bloque elimina el valor y se convierte en un fallo. El reemplazo de contenido no es por tanto un mecanismo de confidencialidad: la política que deba impedir el acceso programático bloquea la llamada o reemplaza el valor.

Los valores canónicos son locales a la ejecución. El agent loop (bucle del agente) persiste `tool/result` solo con `content`, `error` y `meta` opcional; el `tool/code-dispatch` de Code Mode persiste el `content` renderizado de la subllamada y su `isError`. Ningún evento almacena el valor intermedio canónico, de modo que el replay reproduce la presentación pero no puede reconstruir el resultado programático. Cuando una herramienta declara `presentationMeta`, se calcula solo para una llamada de superficie directa; un despacho de Code anidado no recibe metadatos ni tarjeta de resultado. La tarjeta exterior de `run_code` lee en su lugar el contenido final posterior a la política y no declara metadatos de presentación. Las proyecciones de spill genéricas y propiedad de la herramienta omiten de forma similar los despachos anidados, cuyo valor canónico nunca entra en el contexto del modelo.

Las herramientas de primera parte conservan su texto Native existente a la vez que devuelven DTOs de dominio:

| Familia de herramientas | Valor canónico |
|---|---|
| `read` | `{ path, offset, lines: [{ number, text }], totalLines }` |
| `write` | `{ path, operation: "create" | "update", before: string | null, after }` |
| `edit` | `{ path, before, after }` |
| `glob` | `{ paths: string[] }` |
| `grep` | `{ matches: [{ path, lineNumber, line }] }` |
| `web_search` / `web_fetch` | El `WebSearchResult` / `WebFetchResult` normalizado |
| `lsp` | `{ kind: "locations", locations, resolvedWorkspaceUri }` o `{ kind: "hover", hover }` |
| `bash` | `{ kind: "background", jobId }` o `{ kind: "foreground" } & ShellRunResult` |
| `terminal_open` / `terminal_list` / `terminal_send` / `terminal_read` / `terminal_signal` / `terminal_close` | Instantáneas públicas de sesión, DTOs acotados de lectura/envío, resultados de señal/cierre o un handle de job en segundo plano |
| `job_output` / `job_list` / `job_kill` | Instantáneas públicas de tarea sin contabilidad de propietario ni de notificación |
| `subagent` | Handle de job en segundo plano o `{ kind: "foreground", runId, output: JsonValue[] }` |
| `workflow` / `ralph` | `{ runId, agentsStarted, result: JsonValue }` |
| `skill` | `{ name, provider, resourceBase?, content }` |
| `todo_write` | `{ todos, counts }` |
| `ask_user_question` | `{ answers: [{ id, selected, custom? }] }` |
| `exit_plan_mode` | `{ approved: true }` |
| `cordis_inspect` / `cordis_mount` / `cordis_unmount` | Texto de inspección o handles de Plugin temporal tipado |
| `structured_output` | `{ recorded: true }` |
| `run_code` | `{ logs: string[], result?: JsonValue }` |

Los límites de adquisición de provider y ejecutor siguen siendo límites reales del valor canónico. Los límites de solo formato pertenecen a `render`; `glob` y `grep`, por ejemplo, conservan cada ítem adquirido en `value` mientras su proyección Native conserva y vierte con best-effort la primera página configurada. El spill genérico antepone y delega su listener de post-ejecución para que una proyección asíncrona ordinaria propiedad de la herramienta se complete antes del acotado genérico de bytes, independientemente del orden de carga de plugins. Las mutaciones del sistema de archivos derivan metadatos de diff reproducibles de `args` y del valor canónico anterior/posterior en lugar de devolver estado de UI desde el cuerpo.

Los puentes MCP conservan los bloques de protocolo mediante `McpResult<{...}> = { content: JsonValue[]; structuredContent? }`. Un `outputSchema` anunciado se aplica cuando pertenece al subconjunto crudo soportado; los schemas no soportados caen en `JsonValue` en lugar de fingir que los validan. El renderizado Native sigue usando la proyección MCP-a-`ContentBlock` existente, y un `isError` de MCP se convierte en un resultado de herramienta fallido.

## Alternativas consideradas

- **Devolver texto renderizado a Code Mode:** rechazado porque los llamadores seguirían raspando prosa en busca de ids de job, ids de montaje, rutas y resultados de provider estructurados.
- **Persistir los valores canónicos en `tool/result`:** rechazado porque los valores de ejecución anidados no son historia del modelo, no necesitan sobrevivir al replay y crearían un compromiso de formato de sesión y almacenamiento ajeno a la reconstrucción Native.
- **Dejar que las herramientas devuelvan valor y contenido a la vez:** rechazado porque dos resultados propiedad del autor pueden discrepar y la política no puede declarar cuál es autoritativo. El renderizador hace que la presentación sea una proyección determinista del valor validado.
- **Tratar el reemplazo de contenido como redacción de valor:** rechazado porque la presentación y el acceso programático son consumidores distintos; ocultar solo la primera crearía una frontera de seguridad falsa.
- **Exigir salidas de herramienta con raíz de objeto:** rechazado porque los resultados escalares, de array y nulos son APIs JSON legítimas. La raíz de objeto sigue siendo una regla de consumidor para la salida estructurada definida por el llamador de subagente (subagent)/flujo de trabajo.

## Consecuencias

El comportamiento Native y de replay sigue siendo primero-contenido y compatible por bytes, mientras que los llamadores en tiempo de ejecución pueden usar un valor de dominio validado sin analizar ese contenido. Los fallos tienen un mensaje obligatorio más información opcional de clase/código interna, los resultados correctos y fallidos están discriminados, y un resultado fallido nunca puede prometer un valor. Los autores de herramientas deben diseñar el valor y la proyección Native juntos; la declaración extra es intencional porque evita que se infieran contratos programáticos accidentales a partir de la prosa.

Los valores intermedios siguen acotados solo por la capacidad productora y la memoria del proceso. Su omisión del registro significa que el replay no puede recuperarlos, y una política posterior solo-contenido no los oculta. Son propiedades explícitas del contrato local de ejecución, no huecos accidentales.
