# Referencia de autoría de herramientas

[English](adding-a-tool.md) | Español

Referencia de los contratos que debe cumplir una herramienta de cara al modelo. Para una primera herramienta guiada paso a paso, sigue [Crea una herramienta](../user/develop/basic/tool.es.md). `packages/shell/tool-bash` es el ejemplo de tres paquetes con calidad de producción.

## La forma mínima

```ts
import { readFile } from 'node:fs/promises'
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

export const name = 'my-tool'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'read_file',
    description: 'Read a file from disk.',          // what the model sees
    parameters: {
      path: { type: 'string', required: true, description: 'Absolute path' },
      limit: { type: 'number' },                     // optional by default
    },
    output: {
      schema: { type: 'string' },
      render: (_args, value) => [{ type: 'text', text: value }],
    },
    async execute(args, exec) {
      // args is TYPED from the schema: { path: string; limit?: number }
      // exec carries immutable identity + token; signal is the operational field
      return readFile(args.path, { encoding: 'utf8', signal: exec.signal })
    },
  }))
}
```

El registro se basa en efectos: disponer de la fibra del plugin desregistra la herramienta. Los schemas fluyen automáticamente al ensamblaje del system prompt.

## Reglas del contrato de execute()

- **Los args se validan por ti.** `defineTool` valida los `arguments` generados por el modelo contra el `ParameterSchemaSpec` unificado antes de que `execute` se ejecute (tipos, claves requeridas, restricciones literales, uniones de exactamente uno y valores anidados — [validación de args en runtime](../../.agents/notes/implemented/architecture/2026-06-11-runtime-arg-validation.es.md)), así que dentro de `execute` los args coinciden con `InferArgs`. Los nodos de objeto explícitos declaran `additionalProperties: true | false`; la raíz de parámetros implícita permanece abierta. Aun así compruebas a mano las restricciones que el DSL no expresa, como strings no vacíos, números positivos o reglas entre campos. Las herramientas JSON-Schema crudas registradas directamente son dueñas de su propia validación de entrada.
- **El registro toma prestada tu definición de solo lectura.** Una contribución tipada del mismo proceso no es un límite de serialización; no mutes su schema ni reemplaces callbacks después del registro. `schemas()` materializa solo la proyección explícita de cara al modelo. Para hacer hot-swap de una herramienta, dispon de su efecto propietario y registra el reemplazo; el estado mutable dentro del closure del callback sigue siendo estado ordinario de plugin.
- **La identidad de ejecución está protegida.** El registro materializa `arguments` como JSON sin pérdida y desacoplado en una pasada recursiva, congela ese valor antes de que arranque la policy y asigna un `exec.token` opaco; `callId`, `name`, `arguments`, `agent`, `token`, el `signal` requerido propiedad del llamador y un token `parent` opcional del transporte contenedor permanecen inmutables durante el dispatch. `parent` es solo identidad y no expone ninguna ejecución exterior en vivo. Trata `args` como entrada de solo lectura. Solo un wrapper alrededor del dispatch recibe una vista mutable, y puede reemplazar y restaurar el `exec.signal` requerido para imponer un plazo, pero no puede eliminarlo.
- **Declara y devuelve un único valor JSON canónico.** `output.schema` usa `ValueSchemaSpec` y puede tener una raíz de objeto, array, escalar o null. `execute` devuelve solo el valor inferido; el registro lo captura como JSON sin pérdida, lo valida, lo congela y lo pasa a `output.render(args, value)`. No devuelvas bloques de contenido desde el cuerpo ni hagas que los llamadores parseen prosa para extraer ids y campos.
- **Lanzar una excepción o devolver un valor inválido significa `isError`.** El registro captura los throws y contiene los fallos de schema, renderer, proyector de metadatos y JSON sin pérdida antes de que corran los observers. Lanza para fallos de infraestructura. Representa un resultado de dominio exitoso en el valor canónico aunque su renderer Native explique un estado no ideal, como una salida de proceso distinta de cero.
- **Respeta `exec.signal`.** Cancela el trabajo en curso cuando se dispara.
- **Proyecta datos durables de tarjeta con `presentationMeta` (opcional).** `output.presentationMeta(args, value)` deriva JSON reproducible del mismo valor canónico. El core lo persiste en `tool/result` y lo entrega a `presentResult`, así que una tarjeta que necesita hechos del momento del resultado — como los hunks aplicados de `write`/`edit` — sobrevive a la reproducción sin persistir el valor canónico. El proyector se omite para los dispatches de Code anidados porque no tienen tarjetas.
- **Usa `exec.agent` para notificaciones asíncronas.** `agent.inject({ content, source: { kind: 'plugin', plugin: '<name>' } })` añade contexto durable que verá la PRÓXIMA solicitud del modelo — no es un despertador (un agent inactivo sigue inactivo). Protégete contra agents dispuestos (try/catch).

## Trabajo de larga duración

Condiciona `run_in_background` a la config del productor y luego registra a través de `ctx.jobs.start({ kind, label, owner: exec.agent, run })`. El registro rechaza una invocación previamente abortada antes del cuerpo del productor; el runtime valida la propiedad y la disponibilidad del task controller antes de que `run()` empiece el trabajo, y luego aporta el id, el vallado de sesión, las herramientas de control genéricas, los avisos y la limpieza del propietario. Una rama de fondo exitosa devuelve un handle canónico tipado como `{ kind: 'background', jobId }`; su renderer Native puede conservar prosa humana como `started background job bash-1`, pero Code Mode nunca debe parsear esa prosa para recuperar el id.

El productor aporta un `cancel` síncrono, un `done` que no rechaza y se resuelve tras la limpieza de recursos, y un `readOutput` de consumo opcional con formato de salida acotada. Una llamada previamente abortada es un fallo porque no existe ninguna tarea cuyo id pudiera satisfacer el schema de salida exitoso. Una vez que `ctx.jobs.start()` publica el id, usa una señal de cancelación propiedad de la tarea en lugar de `exec.signal`: una cancelación posterior de la llamada exterior deja de esperar a la llamada, pero no mata el trabajo publicado; `job_kill`, la disposición del propietario y el teardown del servicio son dueños de ese ciclo de vida. El trabajo en primer plano sigue acoplado a `exec.signal`. Consulta la [Agent Note del runtime de trabajos en segundo plano](../../.agents/notes/implemented/architecture/2026-06-20-generic-long-running-tool-runtime.es.md) y `dsh-tool-bash` para un productor de streams.

## Policy de ejecución y observación

Prefiere no construir policy de despliegue dentro de la herramienta. Usa `tools/pre-execute` para policy extensible de allow/deny/ask (el [ejemplo de permission gate](extension-cookbook.es.md#a-hook-plugin-permission-gate-example)), `ctx.tools.guard()` para un deny monótono final que los listeners posteriores no pueden deshacer, `tools/execute` para envolver el dispatch con un plazo, reintento o recogida de métricas, `tools/post-execute` para reemplazar el contenido de presentación o el valor devuelto, bloquear el resultado o adjuntar contexto de cara al modelo, y `tools/result` para observar el resultado normalizado e inmutable. Un reemplazo de contenido deja intacto el acceso programático a `value`; la policy de confidencialidad bloquea o reemplaza el valor. Una implementación de sandboxing también puede correr dentro de la implementación del executor de la herramienta; el [README de `dsh-tools`](../../packages/core/tools/README.es.md#extension-points) define las entradas, el orden, los valores de retorno y el comportamiento ante fallos de cada punto de extensión.

## Code Mode llega a tu herramienta sin coste

En [Code Mode](../../packages/core/tools/README.es.md), cada herramienta registrada y visible está disponible como `await tools.<name>(args)` sin integración extra. Los `ToolArgsMap` y `ToolOutputMap` generados derivan los tipos exactos de argumentos y de retorno canónico de los mismos schemas, y las llamadas reentran en el pipeline de ejecución normal. Una llamada exitosa se resuelve al valor JSON canónico final tras la policy, no al contenido Native renderizado. Una llamada fallida rechaza con el `ToolCallError` real; los programas solo pueden inspeccionar su `name`, `toolName` y `message` legible por humanos, no códigos de error internos ni una unión de fallos.

Diseña `output.schema` como una API programática útil: devuelve handles y campos directamente, permite raíces escalar/array/null cuando son el valor honesto y deja la explicación humana en `output.render`. Los valores intermedios son locales a la ejecución, no se persisten ni se truncan para el prompt y no tienen tope de bytes, así que los límites de adquisición veraces del productor y la memoria del proceso siguen importando. Solo los logs/result del `run_code` exterior cruzan el tope de salida configurable y el pipeline de spill de cara al modelo.

## Cómo se renderiza tu herramienta en una UI

El `output.render` de tu herramienta devuelve contenido de cara al modelo; su **tarjeta de UI** es un asunto aparte que se declara mediante proyecciones de presentación puras y los métodos opcionales `presentCall` / `presentResult`. Diseña estos junto al valor canónico. Una herramienta sin presentación de UI cae en una tarjeta genérica (título = nombre de la herramienta, args crudos como entrada).

Ambos métodos devuelven una **intención de renderizado etiquetada con `card`** — elige el tipo de tarjeta que coincida con lo que hace tu herramienta:

- `presentCall(args)` → una `ToolCallView` (la tarjeta PENDING):
  - `{ card: 'generic', title, kind?, rawInput?, content?, locations? }` — el predeterminado. Pon `kind` para un icono (`read`/`search`/…); pon `locations: [{ path, line? }]` para cualquier archivo que toque tu herramienta, para que un editor capaz lo siga o salte a él.
  - `{ card: 'terminal', title, description?, cwd? }` — tu llamada ES un comando de shell. `title` es el comando, `description` se renderiza encima de la tarjeta terminal. (tool-bash.)
  - `{ card: 'diff', title, diffs, locations? }` — tu llamada crea o modifica un archivo. `diffs: [{ path, oldText, newText }]` (`oldText: null` para un archivo nuevo) se renderiza como tarjeta diff en línea. (tool-fs `write`/`edit`.)
- `presentResult(args, { content, isError, meta? })` devuelve la tarjeta completada:
  - `generic` aporta un título y contenido opcionales.
  - `terminal` aporta la salida cruda y metadatos de salida opcionales; cada UI renderiza su vista capaz o de respaldo.
  - `diff` aporta los hunks aplicados, a menudo derivados por `output.presentationMeta` y transportados en el `result.meta` persistido para que la reproducción los repita. Las herramientas de mutación mantienen un resultado diff porque la vista completada reemplaza a la tarjeta pendiente.
  - `search` aporta un resultado de descubrimiento reconstruido desde el `result.meta` persistido: coincidencias agrupadas por archivo (`shape: 'matches'`, grep) o una lista plana de rutas (`shape: 'paths'`, glob), más `truncated`/`total` para que una UI nunca presente un resultado con tope como completo. La vista no lleva texto de resultado (una UI sin tarjeta de búsqueda cae en el contenido crudo del resultado), y no existe una vista de llamada `search` — el estado pendiente de una llamada de descubrimiento sigue siendo una tarjeta genérica, porque las coincidencias solo existen después de `execute`. (tool-fs-search `grep`/`glob`.)
  - `web` aporta una recuperación web completada, discriminada por `kind: 'search' | 'fetch'` (las fuentes de búsqueda estructuradas o el resumen del fetch), derivada de `result.meta`; no lleva texto de cuerpo, así que una UI sin la capacidad `web` cae en el contenido crudo del resultado. (tool-web `web_search`/`web_fetch`.)

Reglas duras (muerden si se rompen):

- **Pureza.** Estos se ejecutan tanto en streaming en vivo como en la REPLAY del log de sesión, así que deben ser funciones puras de `args` (+ el resultado) — NADA de I/O, NADA de leer estado de sesión, NADA de reloj/aleatoriedad. Un diff se deriva de los args (`write` usa `oldText: null` porque un presentador en tiempo de llamada no tiene el contenido previo del archivo); el adaptador de UI, no la herramienta, aporta el contexto de sesión. Si te encuentras queriendo el contenido antiguo del archivo o el directorio de trabajo dentro de `presentCall`, para — eso pertenece a los metadatos durables del resultado o al adaptador, no al presentador.
- **El formato solo de UI queda fuera del resultado del modelo.** Un bloque ` ```console ` con fence, un diff, una ruta relativizada — nada de esto pertenece al valor canónico ni al contenido Native solo para servir a una UI. `output.render` es dueño de la prosa de cara al modelo; `presentationMeta` más los presentadores de tarjeta son dueños del estado de UI reproducible. Una vista de resultado `terminal` lleva la salida cruda y el adaptador añade cualquier marco de respaldo.
- **`defineTool` valida de forma suave la ruta de visualización.** Los argumentos registrados malformados o antiguos hacen que el wrapper devuelva `undefined` (un respaldo genérico) en lugar de lanzar — la visualización nunca debe tumbar una reproducción.

El vocabulario neutral vive en `dsh-tools`; las herramientas nunca importan un tipo de UI o de transporte. Los runtimes host/client mapean cada `card` a su propia vista. El diseño y el porqué están en [la Agent Note de la unión de intenciones de renderizado](../../.agents/notes/implemented/architecture/2026-07-02-tool-render-intent-union.es.md); `dsh-tool-fs` (generic/diff) y `dsh-tool-bash` (terminal) son las implementaciones de referencia.

## Verificación

Sigue la [política de pruebas del repositorio](../testing.es.md) y la documentación de pruebas del paquete propietario. Un cambio publicado visible para el modelo o la UI requiere la cobertura ensamblada que se especifica allí.
