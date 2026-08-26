<!-- Generado por scripts/gen-tool-catalog.ts — no editar a mano.
     Ejecuta `pnpm run gen-tool-catalog` para regenerar. -->

# Catálogo de schemas de herramientas

[English](tool-catalog.md) | Español

Cada herramienta orientada al modelo que un plugin incluido aporta a `ctx.tools`: el `name`, la `description` y los `parameters` de JSON Schema que el modelo recibe a través del ensamblado del system prompt. Complementa las [páginas de subsistema](subsystems/core.es.md) (los tipos más la región de API de Cordis generada de cada página) — esta página son las *herramientas* que se le ofrecen al agent (agente).

Este archivo está GENERADO y su frescura se verifica con `pnpm run verify-tool-catalog` (parte de `doc-sync`) — no lo edites a mano. A diferencia del catálogo de Cordis (una pasada pura sobre el AST del código fuente), este generador ARRANCA cada plugin de herramienta en un contexto real y lee `ctx.tools.schemas()`, porque un schema de herramienta no es estáticamente conocible (enums extendidos en tiempo de ejecución, descripciones concatenadas, nombres guiados por configuración, herramientas MCP de JSON Schema en bruto). Una guarda de completitud hace glob de `packages/*/tool-*` y falla si algún paquete falta en el manifest (manifiesto) de arranque del generador, de modo que una herramienta nueva no puede quedar sin documentar silenciosamente. Consulta [la Agent Note del catálogo de schemas de herramientas](../.agents/notes/implemented/process/2026-07-02-tool-schema-catalog.md).

Ámbito: las herramientas de producto incluidas bajo `packages/*/tool-*`, cada una arrancada con su configuración POR DEFECTO, salvo cuando un campo de Config es OBLIGATORIO sin valor por defecto — ahí el generador debe elegir, y la nota por paquete registra qué rama muestra esta página. El NOMBRE de herramienta registrado puede ser una configuración de tiempo de carga (p. ej. el `toolName` de `tool-subagent`), así que un despliegue puede exponer un paquete bajo un nombre distinto o adicional — una nota por paquete registra esos alias incluidos donde existan. Las herramientas de demostración de `examples/` (p. ej. `echo`) quedan excluidas, en línea con el ámbito de solo paquetes del catálogo de Cordis.

## Mapa de paquetes de herramientas

Esta tabla conecta los nombres de herramienta visibles para el modelo con el paquete de plugin y los seams de servicio que hay detrás. Los JSON Schemas exactos siguen en las secciones por paquete, más abajo.

| Paquete de herramientas | Nombres visibles para el modelo | Requiere | Escribe / afecta | Alias incluidos | Nota de despliegue |
| --- | --- | --- | --- | --- | --- |
| `@deepseek-ai/dsh-tool-ask-user` | `ask_user_question` | `ctx.tools`, `ctx.userQuestions` | `tool/call`, `tool/result after a UI/provider answers the question` | - | ask_user_question detiene la llamada de herramienta hasta que el provider de UI activo devuelve una respuesta humana. |
| `@deepseek-ai/dsh-tools` | `run_code` | `ctx.tools`, `ctx.codeRuntime (execution time)`, `ctx.systemPrompt` | `tool/call`, `one tool/code-dispatch-start + tool/code-dispatch pair per bridged sub-call`, `tool/result` | - | Lo posee el registro de herramientas como transporte reservado fuera de las capas de capacidad filtrables bajo `mode: code` / `mode: both` (consulta la Agent Note de Code Mode). Bajo `code` es la única contribución de cable del registro; las demás capacidades visibles se declaran en una sección SDK generada en el lenguaje del runtime cargado, y un programa las invoca a través de bindings programados bajo el contrato de concurrencia nativo (inicios y política ordenados por envío; los cuerpos seguros ante concurrencia se solapan hasta `maxParallelSubCalls`) que vuelven a entrar en la canalización completa de herramientas con guardas y enlazan cada ejecución anidada con este resultado exterior. |
| `@deepseek-ai/dsh-plan-mode` | `exit_plan_mode` | `ctx.tools`, `ctx.systemPrompt`, `ctx.userQuestions (execution time, opportunistic)` | `tool/call`, `plan/mode inactive on an approved review`, `tool/result` | - | exit_plan_mode permanece en el schema visible para el modelo mientras la planificación está inactiva, para que las transiciones no añadan ruido al catálogo de herramientas además del cambio de política de plan. Su ruta de ejecución rechaza las llamadas fuera del modo plan; en modo plan presenta el plan a través del seam de preguntas al usuario (aprobar / seguir planificando con comentarios), y la aprobación registra el modo plan como inactivo en el límite del paso. |
| `@deepseek-ai/dsh-tool-bash` | `bash` | `ctx.tools`, `ctx.shell`, `ctx.systemPrompt`, `ctx.shellEnv`, `ctx.jobs at call time for run_in_background` | `tool/call`, `tool/result` | - | La herramienta bash es el Consumer visible para el modelo del seam del ejecutor bash. Una ejecución `run_in_background` se registra en el runtime genérico `ctx.jobs` y se recoge/detiene a través de las herramientas `job_*` de `@deepseek-ai/dsh-tool-jobs`; la configuración `enableRunInBackground` (por defecto true) elimina el parámetro por completo cuando está desactivada. |
| `@deepseek-ai/dsh-tool-pwsh` | `pwsh` | `ctx.tools`, `ctx.shell`, `ctx.systemPrompt`, `ctx.shellEnv`, `ctx.jobs at call time for run_in_background` | `tool/call`, `tool/result` | - | La herramienta pwsh es el Consumer del seam del ejecutor bash en dialecto PowerShell para composiciones de Windows (un ejecutor de PowerShell como `@deepseek-ai/dsh-pwsh-local` respalda `ctx.shell`); replica la herramienta bash llamada por llamada salvo los controles de sandbox — las ejecuciones `run_in_background` se registran en el runtime genérico `ctx.jobs` y se recogen/detienen a través de las herramientas `job_*`, y el entorno gestionado `DSH_*` proviene de `@deepseek-ai/dsh-shell-env`. Cada llamada se ejecuta en un proceso nuevo (sin sesión PTY persistente), con rutas nativas `C:\...` y variables `$env:NAME`. |
| `@deepseek-ai/dsh-tool-cordis` | `cordis_define`, `cordis_inspect_list`, `cordis_inspect_query`, `cordis_inspect_self`, `cordis_run`, `cordis_stop`, `cordis_undefine` | `ctx.tools`, `ctx.dynamicCordisRunner` | `tool/call`, `tool/result`, `process-local dynamic package lifecycle` | - | No está en ningún árbol incluido (un opt-in deliberado: el código de paquetes dinámicos llega al runtime real; consulta .agents/notes/implemented/feature/2026-07-08-self-referential-cordis-toolset.md). El conjunto de herramientas inyecta `ctx.dynamicCordisRunner` desde `@deepseek-ai/dsh-cordis-host-runner`, que posee el registro de definiciones y el sandbox de vm; una composición sin él nunca activa las herramientas. Un paquete en ejecución puede registrar herramientas ADICIONALES visibles para el modelo hasta que se detenga, se elimine o DSH se reinicie; una cabecera completa de solicitud de cambio registra esos cambios del conjunto de herramientas. |
| `@deepseek-ai/dsh-tool-bash-persistent` | `bash` | `ctx.tools`, `ctx.terminals`, `an owning Agent at execution time` | `tool/call`, `PTY shell state`, `tool/result` | - | Una herramienta bash persistente aislada por propietario; la composición de despliegue suministra el backend PTY y puede sobrescribir la descripción del entorno visible para el modelo. |
| `@deepseek-ai/dsh-tool-pwsh-persistent` | `pwsh` | `ctx.tools`, `ctx.terminals`, `an owning Agent at execution time` | `tool/call`, `PTY shell state`, `tool/result` | - | Una herramienta pwsh persistente aislada por propietario, la contraparte de Windows de la herramienta bash persistente; la composición de despliegue suministra un backend PTY en dialecto pwsh y puede sobrescribir la descripción del entorno visible para el modelo. |
| `@deepseek-ai/dsh-tool-str-replace-editor` | `str_replace_editor` | `ctx.tools`, `ctx.fs` | `tool/call`, `fs/observed after view presence/absence, edit absence, or successful mutation`, `tool/result` | - | Herramienta independiente de ver/crear/reemplazo literal único/inserción de líneas sobre el seam del sistema de archivos; se compone con cualquier API de shell o terminal. |
| `@deepseek-ai/dsh-tool-fs` | `edit`, `read`, `read_image`, `write` | `ctx.tools`, `ctx.fs`, `ctx.systemPrompt`, `ctx.attachments (image-tool registration)`, `ctx.llm + an image-capable route (image-tool execution)` | `tool/call`, `fs/write-intent or fs/edit-intent for mutations`, `fs/observed after read presence/absence or successful file operation`, `durable attachment (read_image)`, `tool/result` | - | La política de leer-antes-de-escribir/editar la añade `@deepseek-ai/dsh-fs-observation-policy` (un plugin de compuerta de eventos `fs/*`, sin cambio de schema); se espera que un despliegue que carga estas herramientas también la cargue. La herramienta de imagen no se registra sin `ctx.attachments`; su schema es independiente de la ruta, y la ejecución se niega salvo que el modelo enrutado exacto declare entrada de imagen. |
| `@deepseek-ai/dsh-tool-fs-search` | `glob`, `grep` | `ctx.tools`, `ctx.subprocess`, `ctx.systemPrompt` | `tool/call`, `tool/result` | - | glob y grep son herramientas de descubrimiento incondicionales que lanzan el binario ripgrep empaquetado (`@vscode/ripgrep`) a través de ctx.subprocess como llamadas normales en primer plano (nunca trabajos en segundo plano) — sin instalación de `rg` en el host y sin capa de shell. El catálogo usa `sampleOverCapGlobResults: true`; los despliegues deben elegir ese comportamiento explícitamente. Los resultados con tope guardan la lista completa formateada a través del backend opcional ctx.spillStore; los localizadores devueltos se pueden leer/buscar después cuando el backend expone rutas locales en despliegues colocalizados. |
| `@deepseek-ai/dsh-tool-terminal` | `terminal_close`, `terminal_list`, `terminal_open`, `terminal_read`, `terminal_send`, `terminal_signal` | `ctx.tools`, `ctx.terminals`, `ctx.systemPrompt`, `ctx.jobs at call time for run_in_background` | `tool/call`, `tool/result` | - | Las seis herramientas de terminal son opt-in y complementan las herramientas de shell/sistema de archivos de un solo uso. `terminal_send(run_in_background: true)` se registra en `ctx.jobs`; TUI, secuencias de teclas con nombre, BEL, redimensionado, auto-inicio y compartición entre agents están ausentes del schema. |
| `@deepseek-ai/dsh-tool-goal` | `create_goal`, `get_goal`, `update_goal` | `ctx.tools`, `ctx.agents`, `ctx.goals`, `ctx.systemPrompt`, `a calling Agent in an authorized open turn` | `tool/call`, `goal/change for mutations`, `tool/result` | - | create, edit, pause y resume requieren autoridad raíz humana directa; complete y blocked también aceptan la ronda actual exacta del objetivo. El límite inferior por defecto de blocked es de tres rondas admitidas. |
| `@deepseek-ai/dsh-schedule` | `schedule_create`, `schedule_delete`, `schedule_list` | `ctx.tools`, `ctx.sessions`, `Session persistence`, `a future live root Agent` | `tool/call`, `schedule/change create or delete`, `tool/result` | - | Solo se registra dentro de ámbitos de Agent raíz en vivo creados después de que cargue el plugin de Schedule opt-in. La versión 1 acepta after_seconds, at absoluto explícito y every_seconds de tasa fija acotado, y divulga la entrega local a la sesión; las lecturas y mutaciones de gestión requieren la barrera compartida de persistencia de sesión. |
| `@deepseek-ai/dsh-tool-lsp` | `lsp` | `ctx.tools`, `ctx.lsp`, `ctx.systemPrompt` | `tool/call`, `tool/result` | - | La herramienta lsp mantiene la selección de provider y los subprocesos del language server detrás de ctx.lsp, por lo que su schema visible para el modelo permanece estable entre providers. Requiere un provider registrado (p. ej. `@deepseek-ai/dsh-lsp-stdio`) en tiempo de ejecución; sin uno, una consulta devuelve el error estructurado `LSP_UNAVAILABLE` en lugar de cambiar el schema. |
| `@deepseek-ai/dsh-tool-ralph` | `ralph` | `ctx.tools`, `ctx.workflowEngine`, `ctx.subagents`, `ctx.systemPrompt`, `a calling Agent (exec.agent parents every fresh round)` | `tool/call`, `tool/result`, `workflow and child session events during execution` | - | Un flujo de trabajo fijo en primer plano inicia un hijo estructurado nuevo por ronda; el modelo solo selecciona el objetivo inmutable y un tope de rondas opcional. |
| `@deepseek-ai/dsh-tool-skill` | `skill` | `ctx.tools`, `ctx.agents`, `ctx.skills` | `tool/call`, `tool/result`, `user/message replacement catalogs via agent.inject()` | - | - |
| `@deepseek-ai/dsh-tool-session-query` | `session_event_read`, `session_event_search`, `session_event_trace`, `session_search`, `session_trace` | `ctx.tools`, `ctx.systemPrompt`, `ctx.sessionQuery`, `a calling Agent for workspace authority` | `tool/call`, `tool/result` | - | Las cinco herramientas de solo lectura ocultan los cursores de los providers y autorizan cada resultado desde la sesión inmutable del agent que llama. El paquete es opt-in; las composiciones que necesitan plazos exigidos o salida en línea acotada también montan las políticas genéricas de timeout o spill. |
| `@deepseek-ai/dsh-tool-subagent` | `subagent` | `ctx.tools`, `ctx.subagents`, `ctx.systemPrompt` | `tool/call`, `tool/result`, `child session events through the chosen provider` | `subagent`, `subagent_fork` | El nombre de herramienta registrado es la configuración de tiempo de carga `toolName` (por defecto `subagent`); el schema anterior es ese valor por defecto. Las composiciones incluidas cargan este paquete una vez por backend de subagente (subagent), por lo que el modelo además ve `subagent_fork` enlazado al backend de fork. La descripción de cada instancia, el parámetro `run_in_background` y la política de system prompt siguen su propio `backgroundMode` y `enableRunInBackground`, así que los dos schemas incluidos no son idénticos: `subagent` es `continuable` y por defecto envía las llamadas omitidas al fondo con entrega automática de liquidación, mientras que `subagent_fork` sigue siendo `one-shot` y por defecto las envía al primer plano — consulta `packages/bundle/base/cordis.patch.yml` y `examples/acp-agent/cordis.yml`. |
| `@deepseek-ai/dsh-tool-subagent-control` | `interrupt_agent`, `list_agents`, `send_message` | `ctx.tools`, `ctx.subagents`, `ctx.agents and ctx.sessionProjections (list_agents only)` | `tool/call`, `tool/result`, `child session events through ctx.subagents` | - | Las herramientas de control con nombre global sobre subagentes de fondo continuables: las instancias de `tool-subagent` enlazadas a un provider registran herramientas de delegación distintas, mientras que este paquete registra `send_message` e `interrupt_agent` una sola vez, además de `list_agents` desde su plugin `/list-agents` cargado por separado (cuyas filas del catálogo usan los registros sessionProjections y de Agent en vivo). |
| `@deepseek-ai/dsh-tool-subagent-report` | `report` | `ctx.subagents`, `ctx.systemPrompt`, `a live continuable in-process child Agent` | `tool/call`, `tool/result`, `a user-role message in the direct parent session` | - | Se registra por hijo continuable en proceso en lugar de globalmente, por lo que este schema solo es visible dentro de ese hijo y sobrevive a su `toolFilter` global. La misma contribución instala la sección de prompt `tool:report` con ámbito de hijo, que este catálogo no renderiza. La herramienta `send_message` orientada al padre se instala de forma independiente. |
| `@deepseek-ai/dsh-tool-jobs` | `job_kill`, `job_list`, `job_output` | `ctx.tools`, `ctx.jobs`, `ctx.systemPrompt` | `tool/call`, `tool/result`, `user/message via agent.inject() for background completion notices` | - | El controlador de trabajos en segundo plano agnóstico al tipo: los comandos bash en segundo plano, los envíos PTY y los subagentes se leen, listan y matan con las mismas tres herramientas. Cargar el plugin adjunta el controlador que arma el `ctx.jobs.start()` de los productores. |
| `@deepseek-ai/dsh-experimental-tool-agent-team` | `followup_task`, `interrupt_agent`, `list_agents`, `send_message`, `spawn_teammate`, `team_task_create`, `team_task_get`, `team_task_list`, `team_task_update`, `wait_agent` | `ctx.tools`, `ctx.systemPrompt`, `ctx.agentTeams`, `an exact live Team member Agent` | `tool/call`, `team/member`, `team/message/queued`, `team/message/delivered`, `team/task`, `tool/result` | - | Las diez herramientas tienen ámbito para Team Leads implícitos y compañeros de equipo durables. El bundle dsh-base incluido mantiene el paquete desactivado; el parche de perfil Agent Teams documentado lo activa mientras desactiva los nombres de control heredados de hijos continuables. |
| `@deepseek-ai/dsh-tool-todo` | `todo_write` | `ctx.tools`, `owning Agent session` | `tool/call`, `todo/write`, `tool/result` | - | todo_write es estado propiedad de la sesión; las UIs renderizan el último evento todo/write como una lista de verificación. `allowParallelInProgress` es obligatorio y no tiene valor por defecto, así que el catálogo declara su elección: `true`, cuya descripción invita a varios elementos `in_progress`. Un despliegue que elige `false` recibe la misma herramienta con una descripción que pide exactamente una tarea activa. |
| `@deepseek-ai/dsh-tool-workflow` | `workflow` | `ctx.tools`, `ctx.workflowEngine`, `ctx.systemPrompt`, `a calling Agent (exec.agent parents the script children)` | `tool/call`, `tool/result` | - | - |
| `@deepseek-ai/dsh-tool-web` | `web_fetch`, `web_search` | `ctx.tools`, `ctx.web`, `ctx.systemPrompt` | `tool/call`, `tool/result` | - | web_search y web_fetch mantienen la selección de provider detrás de ctx.web para que los schemas visibles para el modelo permanezcan estables entre cambios de backend. |

<a id="deepseek-aidsh-tool-ask-user"></a>

## `@deepseek-ai/dsh-tool-ask-user`

### `ask_user_question`

Haz al usuario una pregunta concisa cuando necesites confirmación, una elección o información que falte antes de continuar. Envía una o más preguntas, cada una con un id estable que se repetirá en la respuesta.

```json
{
  "type": "object",
  "properties": {
    "questions": {
      "type": "array",
      "description": "Questions to ask the user before continuing.",
      "items": {
        "type": "object",
        "additionalProperties": true,
        "properties": {
          "id": {
            "type": "string",
            "description": "Stable id for this question; echoed in the answer."
          },
          "question": {
            "type": "string",
            "description": "The specific question to ask the user."
          },
          "header": {
            "type": "string",
            "description": "Optional short heading for the question, such as \"Confirm\" or \"Choose Mode\"."
          },
          "options": {
            "type": "array",
            "description": "Optional choices to show the user. If you recommend one, put it first and append \"(Recommended)\" to that label.",
            "items": {
              "type": "object",
              "additionalProperties": true,
              "properties": {
                "label": {
                  "type": "string",
                  "description": "Short user-facing option label."
                },
                "description": {
                  "type": "string",
                  "description": "One sentence explaining the tradeoff or impact."
                }
              },
              "required": [
                "label"
              ]
            }
          },
          "multi_select": {
            "type": "boolean",
            "description": "Whether the user may select more than one option. Defaults to false."
          }
        },
        "required": [
          "id",
          "question"
        ]
      }
    }
  },
  "required": [
    "questions"
  ]
}
```


Fuente: [`packages/interaction/tool-ask-user/src/index.ts`](../packages/interaction/tool-ask-user/src/index.ts)

ask_user_question detiene la llamada de herramienta hasta que el provider de UI activo devuelve una respuesta humana.

<a id="deepseek-aidsh-tools"></a>

## `@deepseek-ai/dsh-tools`

### `run_code`

Ejecuta un programa TypeScript contra las herramientas disponibles. Toma dos argumentos obligatorios: `code`, el CUERPO de una función async (solo sintaxis borrable; el `await` y el `return` de nivel superior funcionan), y `description`, un resumen breve de lo que hace el programa. Llama a las herramientas como `await tools.name(args)` según las declaraciones del system prompt. Solo lo que imprimes o devuelves es salida del programa — depúrala. Los resultados de subtools con imágenes se adjuntan después de la ejecución.

```json
{
  "type": "object",
  "properties": {
    "code": {
      "type": "string",
      "description": "The program: the body of an async TypeScript function."
    },
    "description": {
      "type": "string",
      "description": "Clear, concise description of what this program does in active voice, 5-10 words (shown in the UI). Examples: \"Count TODO markers across packages\"; \"Read failing test and its fixture\"; \"Rename config key in every cordis.yml\"."
    }
  },
  "required": [
    "code",
    "description"
  ]
}
```


Fuente: [`packages/core/tools/src/code-mode.ts`](../packages/core/tools/src/code-mode.ts)

Lo posee el registro de herramientas como transporte reservado fuera de las capas de capacidad filtrables bajo `mode: code` / `mode: both` (consulta la Agent Note de Code Mode). Bajo `code` es la única contribución de cable del registro; las demás capacidades visibles se declaran en una sección SDK generada en el lenguaje del runtime cargado, y un programa las invoca a través de bindings programados bajo el contrato de concurrencia nativo (inicios y política ordenados por envío; los cuerpos seguros ante concurrencia se solapan hasta `maxParallelSubCalls`) que vuelven a entrar en la canalización completa de herramientas con guardas y enlazan cada ejecución anidada con este resultado exterior.

<a id="deepseek-aidsh-plan-mode"></a>

## `@deepseek-ai/dsh-plan-mode`

### `exit_plan_mode`

Úsalo solo en modo plan. Presenta tu plan para que el usuario lo revise y, al aprobarlo, sal del modo plan. Envía el plan COMPLETO como markdown, empezando por un encabezado # que lo nombre. El usuario puede aprobar (lleva a cabo el plan a partir de tu siguiente paso) o seguir planificando — su comentario vuelve en el resultado de la herramienta; revísalo y vuelve a presentarlo.

```json
{
  "type": "object",
  "properties": {
    "plan": {
      "type": "string",
      "description": "The complete plan, as markdown, starting with a # heading that names it."
    }
  },
  "required": [
    "plan"
  ]
}
```


Fuente: [`packages/plan/plan-mode/src/index.ts`](../packages/plan/plan-mode/src/index.ts)

exit_plan_mode permanece en el schema visible para el modelo mientras la planificación está inactiva, para que las transiciones no añadan ruido al catálogo de herramientas además del cambio de política de plan. Su ruta de ejecución rechaza las llamadas fuera del modo plan; en modo plan presenta el plan a través del seam de preguntas al usuario (aprobar / seguir planificando con comentarios), y la aprobación registra el modo plan como inactivo en el límite del paso.

<a id="deepseek-aidsh-tool-bash"></a>

## `@deepseek-ai/dsh-tool-bash`

### `bash`

Ejecuta un comando bash (`bash -c`) y devuelve su stdout/stderr. Cada llamada se ejecuta en un shell nuevo: ningún estado (cwd, variables, funciones) persiste entre llamadas — pasa `workdir` en lugar de usar `cd`. Las salidas con código distinto de cero se notifican como `[exit code: N]`. Los datos actuales del entorno del harness se exponen a través de las variables gestionadas `$DSH_*`; inspecciónalas cuando lo necesites. Los comandos pueden ejecutarse bajo un sandbox de archivos; una operación de archivo bloqueada se notifica como `[sandbox: file access denied under <mode> mode]` — una denegación de política, no un fallo del comando; no lo reintentes de otra forma. La salida larga se trunca a su final; la salida completa se guarda en un archivo cuya ruta se notifica cuando está disponible. Pon `run_in_background: true` para comandos de larga duración: la llamada devuelve un id de trabajo de inmediato; lee su salida con `job_output` y detenlo con `job_kill`.

```json
{
  "type": "object",
  "properties": {
    "command": {
      "type": "string",
      "description": "The bash command to execute."
    },
    "description": {
      "type": "string",
      "description": "Clear, concise description of what this command does in active voice, 5-10 words (shown in the UI). Examples: \"ls\" → \"List files in current directory\"; \"git status\" → \"Show working tree status\"; \"npm install\" → \"Install package dependencies\"."
    },
    "timeoutMs": {
      "type": "number",
      "description": "Timeout in milliseconds. The executor applies its configured default and cap, and kills the command on expiry."
    },
    "workdir": {
      "type": "string",
      "description": "Working directory for this command. Defaults to the session workspace; a relative path is resolved against it."
    },
    "run_in_background": {
      "type": "boolean",
      "description": "Run in the background and return a job id immediately (collect with job_output, stop with job_kill). No timeout applies."
    }
  },
  "required": [
    "command",
    "description"
  ]
}
```


Fuente: [`packages/shell/tool-bash/src/index.ts`](../packages/shell/tool-bash/src/index.ts)

La herramienta bash es el Consumer visible para el modelo del seam del ejecutor bash. Una ejecución `run_in_background` se registra en el runtime genérico `ctx.jobs` y se recoge/detiene a través de las herramientas `job_*` de `@deepseek-ai/dsh-tool-jobs`; la configuración `enableRunInBackground` (por defecto true) elimina el parámetro por completo cuando está desactivada.

<a id="deepseek-aidsh-tool-pwsh"></a>

## `@deepseek-ai/dsh-tool-pwsh`

### `pwsh`

Ejecuta un comando PowerShell (`pwsh -Command`) y devuelve su stdout/stderr. Cada llamada se ejecuta en un proceso pwsh nuevo: ningún estado (cwd, variables, funciones) persiste entre llamadas — pasa `workdir` en lugar de usar `cd`. Las rutas usan la forma nativa de Windows (`C:\...`); lee las variables de entorno con `$env:NAME`. Las salidas con código distinto de cero se notifican como `[exit code: N]`. Los datos actuales del entorno del harness se exponen a través de las variables gestionadas `$env:DSH_*`; inspecciónalas cuando lo necesites. Los comandos pueden ejecutarse bajo un sandbox de archivos; una operación de archivo bloqueada se notifica como `[sandbox: file access denied under <mode> mode]` — una denegación de política, no un fallo del comando; no lo reintentes de otra forma. La salida larga se trunca a su final; la salida completa se guarda en un archivo cuya ruta se notifica cuando está disponible. En Windows, un comando forzado a terminar se liquida como `[exit code: 1]` sin marcador de señal — trátalo como una interrupción, no como un fallo del comando. Pon `run_in_background: true` para comandos de larga duración: la llamada devuelve un id de trabajo de inmediato; lee su salida con `job_output` y detenlo con `job_kill`.

```json
{
  "type": "object",
  "properties": {
    "command": {
      "type": "string",
      "description": "The PowerShell command to execute."
    },
    "description": {
      "type": "string",
      "description": "Clear, concise description of what this command does in active voice, 5-10 words (shown in the UI). Examples: \"ls\" → \"List files in current directory\"; \"git status\" → \"Show working tree status\"; \"Get-Process\" → \"List running processes\"."
    },
    "timeoutMs": {
      "type": "number",
      "description": "Timeout in milliseconds. The executor applies its configured default and cap, and kills the command on expiry."
    },
    "workdir": {
      "type": "string",
      "description": "Working directory for this command. Defaults to the session workspace; a relative path is resolved against it."
    },
    "run_in_background": {
      "type": "boolean",
      "description": "Run in the background and return a job id immediately (collect with job_output, stop with job_kill). No timeout applies."
    }
  },
  "required": [
    "command",
    "description"
  ]
}
```


Fuente: [`packages/shell/tool-pwsh/src/index.ts`](../packages/shell/tool-pwsh/src/index.ts)

La herramienta pwsh es el Consumer del seam del ejecutor bash en dialecto PowerShell para composiciones de Windows (un ejecutor de PowerShell como `@deepseek-ai/dsh-pwsh-local` respalda `ctx.shell`); replica la herramienta bash llamada por llamada salvo los controles de sandbox — las ejecuciones `run_in_background` se registran en el runtime genérico `ctx.jobs` y se recogen/detienen a través de las herramientas `job_*`, y el entorno gestionado `DSH_*` proviene de `@deepseek-ai/dsh-shell-env`. Cada llamada se ejecuta en un proceso nuevo (sin sesión PTY persistente), con rutas nativas `C:\...` y variables `$env:NAME`.

<a id="deepseek-aidsh-tool-cordis"></a>

## `@deepseek-ai/dsh-tool-cordis`

### `cordis_define`

Define un Package de Cordis inmutable. Para un Plugin nuevo, usa kind:"new" y proporciona solo un prefijo semántico de 3–6 letras minúsculas en inglés; el Host devuelve el pluginId y el packageId finales. Para modificar un Plugin existente, usa kind:"existing" con su pluginId exacto para añadir un Package sin sobrescribir versiones anteriores. Proporciona al menos uno de code.host y code.client. Cada valor es un cuerpo de función JavaScript plano que devuelve un Plugin de Cordis; no se realiza ninguna transformación de TypeScript, JSX ni imports. Consulta Inspect antes de depender de un Service, Event, Builtin, Slot o token. Define solo valida parámetros y sintaxis y registra el origen: no solicita aprobación, no ejecuta apply ni cambia currentPackageId. En caso de éxito, llama a cordis_run con los IDs devueltos.

```json
{
  "type": "object",
  "properties": {
    "plugin": {
      "oneOf": [
        {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "kind": {
              "type": "string",
              "const": "new"
            },
            "idPrefix": {
              "type": "string",
              "description": "Suggested semantic prefix of 3–6 lowercase English letters; the Host adds a unique numeric suffix."
            }
          },
          "required": [
            "kind",
            "idPrefix"
          ]
        },
        {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "kind": {
              "type": "string",
              "const": "existing"
            },
            "pluginId": {
              "type": "string",
              "description": "Exact ID of an existing Plugin; the new Package is appended to that instance."
            }
          },
          "required": [
            "kind",
            "pluginId"
          ]
        }
      ]
    },
    "name": {
      "type": "string",
      "description": "Short, readable Package name."
    },
    "purpose": {
      "type": "string",
      "description": "One-sentence, user-facing description of the Package purpose."
    },
    "code": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "host": {
          "type": "string",
          "description": "Plain JavaScript function body that returns the Host-half Cordis Plugin."
        },
        "client": {
          "type": "string",
          "description": "Plain JavaScript function body that returns the browser Client-half Cordis Plugin."
        }
      }
    }
  },
  "required": [
    "plugin",
    "name",
    "purpose",
    "code"
  ]
}
```


Fuente: [`packages/extensions/tool-cordis/src/index.ts`](../packages/extensions/tool-cordis/src/index.ts)

### `cordis_inspect_list`

Lista cada Provider de Inspect de Cordis conocido actualmente por el Host, incluidos los Providers locales del Host y los últimos manifests sincronizados desde el Client. Cada entrada incluye su plataforma, propósito, métodos de solo lectura y schemas de entrada/salida. Llama a esta Tool antes de crear o modificar un Package y, después, selecciona el provider y el método para cordis_inspect_query a partir de su resultado. No adivines nombres ni trates un método de Inspect como un Service de negocio que el código del Plugin pueda llamar.

```json
{
  "type": "object",
  "properties": {}
}
```


Fuente: [`packages/extensions/tool-cordis/src/index.ts`](../packages/extensions/tool-cordis/src/index.ts)

### `cordis_inspect_query`

Ejecuta una consulta de solo lectura declarada explícitamente por un Provider de Inspect. platform, provider y method deben venir de cordis_inspect_list, y input debe cumplir el schema de ese método. Usa esta Tool antes de cordis_define para leer métodos exactos de Service, modos de Event, firmas de Builtin, schemas de Tool, tokens de tema o árboles y props de Slot en vivo. Las consultas del Host se ejecutan localmente. Una consulta del Client espera la primera respuesta de página válida y permanece pendiente hasta que una página responde o la Tool se cancela. Esta Tool no puede invocar métodos de Service de negocio ni modificar el runtime. Para Service.listService y Event.listEvents, consulta sin input para navegar por el directorio compacto de firmas y, después, consulta el service o event exacto para su contrato estructurado y los tipos referenciados. Para Slots.listSubTree, consulta sin root para navegar por el árbol compacto y, después, consulta el root exacto para su contrato de registro completo y sus props.

```json
{
  "type": "object",
  "properties": {
    "platform": {
      "type": "string",
      "description": "Runtime platform that owns the Provider.",
      "enum": [
        "host",
        "client"
      ]
    },
    "provider": {
      "type": "string",
      "description": "Exact Provider ID returned by cordis_inspect_list."
    },
    "method": {
      "type": "string",
      "description": "Exact method name declared by the Provider manifest."
    },
    "input": {
      "description": "Optional query input; it must satisfy the method input schema."
    }
  },
  "required": [
    "platform",
    "provider",
    "method"
  ]
}
```


Fuente: [`packages/extensions/tool-cordis/src/index.ts`](../packages/extensions/tool-cordis/src/index.ts)

### `cordis_inspect_self`

Inspecciona los objetos Cordis dinámicos propiedad de la Session actual con niveles de detalle crecientes. Sin IDs, lista solo resúmenes de Plugin. Con solo pluginId, devuelve los punteros de versión, el último Run y el resumen de cada Package. Solo pluginId más packageId devuelve el código fuente Host/Client de ese Package inmutable y los diagnósticos de runtime. packageId no puede proporcionarse solo. Consulta un Package exacto antes de gestionar @pluginId, reparar un fallo asíncrono o definir una versión actualizada. Esta Tool es de solo lectura: no ejecuta código ni cambia punteros de versión.

```json
{
  "type": "object",
  "properties": {
    "pluginId": {
      "type": "string",
      "description": "Stable Plugin ID returned by cordis_define or injected by @pluginId; omit it to list every current Plugin."
    },
    "packageId": {
      "type": "string",
      "description": "Exact immutable Package ID owned by pluginId; when specified, source and diagnostics are returned."
    }
  }
}
```


Fuente: [`packages/extensions/tool-cordis/src/index.ts`](../packages/extensions/tool-cordis/src/index.ts)

### `cordis_run`

Activa un Package exacto de un Plugin dinámico. Usa mode:"run" para la primera activación, reiniciar currentPackageId o hacer rollback. Cuando existe un current, usa mode:"update" para cambiar a un Package distinto, aunque el Plugin esté detenido. Un Package del Client no autorizado crea una solicitud de aprobación y devuelve awaiting-approval; un Package autorizado devuelve starting y continúa de forma asíncrona en el navegador. Ninguno de los dos resultados espera el desenlace final dentro de la Tool. currentPackageId solo cambia tras un éxito completo; en caso de fallo, el current anterior y el next objetivo permanecen. El éxito asíncrono, el rechazo o el fallo técnico se notifican a través del estado y del steering (direccionamiento). Tras un fallo técnico, lee los diagnósticos con cordis_inspect_self, corrige el mismo Plugin y reintenta de forma autónoma. No vuelvas a solicitar la aprobación después de que el usuario la rechace.

```json
{
  "type": "object",
  "properties": {
    "pluginId": {
      "type": "string",
      "description": "Stable Plugin ID returned by cordis_define."
    },
    "packageId": {
      "type": "string",
      "description": "Exact immutable Package ID to activate under that Plugin."
    },
    "mode": {
      "type": "string",
      "description": "Use run for the first activation, restarting current, or rollback; use update to switch from current to a different Package.",
      "enum": [
        "run",
        "update"
      ]
    }
  },
  "required": [
    "pluginId",
    "packageId",
    "mode"
  ]
}
```


Fuente: [`packages/extensions/tool-cordis/src/index.ts`](../packages/extensions/tool-cordis/src/index.ts)

### `cordis_stop`

Detén el Run actual de un Plugin dinámico y cancela las solicitudes de aprobación o activación sin terminar. Conserva el Plugin, cada Package inmutable, los grants, currentPackageId y nextPackageId para que pueda ejecutarse o actualizarse directamente más tarde. Detener un Plugin ya detenido tiene éxito de forma idempotente. Usa esta Tool para desactivar efectos temporalmente; usa cordis_undefine para la eliminación permanente.

```json
{
  "type": "object",
  "properties": {
    "pluginId": {
      "type": "string",
      "description": "Stable dynamic Plugin ID to stop."
    }
  },
  "required": [
    "pluginId"
  ]
}
```


Fuente: [`packages/extensions/tool-cordis/src/index.ts`](../packages/extensions/tool-cordis/src/index.ts)

### `cordis_undefine`

Elimina de forma permanente un Plugin dinámico propiedad de la Session actual. Si está en ejecución o a la espera de aprobación, primero detenlo y cancela la solicitud, y después elimina cada Package, grant y puntero de versión. Cuando esto devuelve, su pluginId, packageIds, la referencia @ y las vistas de negocio del Package quedan inválidas; las tarjetas históricas conservan solo un registro "Plugin removed". No llames a esta Tool cuando las versiones deban permanecer disponibles para reiniciar o hacer rollback; usa cordis_stop en su lugar.

```json
{
  "type": "object",
  "properties": {
    "pluginId": {
      "type": "string",
      "description": "Stable dynamic Plugin ID to remove permanently."
    }
  },
  "required": [
    "pluginId"
  ]
}
```


Fuente: [`packages/extensions/tool-cordis/src/index.ts`](../packages/extensions/tool-cordis/src/index.ts)

No está en ningún árbol incluido (un opt-in deliberado: el código de paquetes dinámicos llega al runtime real; consulta .agents/notes/implemented/feature/2026-07-08-self-referential-cordis-toolset.md). El conjunto de herramientas inyecta `ctx.dynamicCordisRunner` desde `@deepseek-ai/dsh-cordis-host-runner`, que posee el registro de definiciones y el sandbox de vm; una composición sin él nunca activa las herramientas. Un paquete en ejecución puede registrar herramientas ADICIONALES visibles para el modelo hasta que se detenga, se elimine o DSH se reinicie; una cabecera completa de solicitud de cambio registra esos cambios del conjunto de herramientas.

<a id="deepseek-aidsh-tool-bash-persistent"></a>

## `@deepseek-ai/dsh-tool-bash-persistent`

### `bash`

Ejecuta comandos en un shell bash persistente. El estado, incluidos el directorio actual y las variables de entorno exportadas, persiste entre llamadas para este agent.

```json
{
  "type": "object",
  "properties": {
    "command": {
      "type": "string",
      "description": "The bash command to run. Relative path is preferred in the command."
    }
  },
  "required": [
    "command"
  ]
}
```


Fuente: [`packages/shell/tool-bash-persistent/src/index.ts`](../packages/shell/tool-bash-persistent/src/index.ts)

Una herramienta bash persistente aislada por propietario; la composición de despliegue suministra el backend PTY y puede sobrescribir la descripción del entorno visible para el modelo.

<a id="deepseek-aidsh-tool-pwsh-persistent"></a>

## `@deepseek-ai/dsh-tool-pwsh-persistent`

### `pwsh`

Ejecuta comandos en un shell PowerShell persistente. El estado, incluidos el directorio actual y las variables de entorno exportadas, persiste entre llamadas para este agent.

```json
{
  "type": "object",
  "properties": {
    "command": {
      "type": "string",
      "description": "The PowerShell command to run. Relative path is preferred in the command."
    }
  },
  "required": [
    "command"
  ]
}
```


Fuente: [`packages/shell/tool-pwsh-persistent/src/index.ts`](../packages/shell/tool-pwsh-persistent/src/index.ts)

Una herramienta pwsh persistente aislada por propietario, la contraparte de Windows de la herramienta bash persistente; la composición de despliegue suministra un backend PTY en dialecto pwsh y puede sobrescribir la descripción del entorno visible para el modelo.

<a id="deepseek-aidsh-tool-str-replace-editor"></a>

## `@deepseek-ai/dsh-tool-str-replace-editor`

### `str_replace_editor`

Herramienta de edición personalizada para ver, crear y editar archivos
* El estado persiste entre las llamadas a comandos y las conversaciones con el usuario
* Si `path` es un archivo, `view` muestra el resultado de aplicar `cat -n`. Si `path` es un directorio, `view` lista los archivos y directorios no ocultos hasta 2 niveles de profundidad
* El comando `create` no se puede usar si el `path` especificado ya existe como archivo
* Si un `command` genera una salida larga, se truncará y se marcará con `<response clipped>`

Notas para usar el comando `str_replace`:
* El parámetro `old_str` debe coincidir EXACTAMENTE con una o más líneas consecutivas del archivo original. ¡Cuidado con los espacios en blanco!
* Si el parámetro `old_str` no es único en el archivo, no se realizará el reemplazo. Asegúrate de incluir suficiente contexto en `old_str` para hacerlo único
* El parámetro `new_str` debe contener las líneas editadas que reemplazarán a `old_str`

```json
{
  "type": "object",
  "properties": {
    "command": {
      "type": "string",
      "description": "The commands to run. Allowed options are: `view`, `create`, `str_replace`, `insert`.",
      "enum": [
        "view",
        "create",
        "str_replace",
        "insert"
      ]
    },
    "path": {
      "type": "string",
      "description": "Absolute path to file or directory, e.g. `/repo/file.py` or `/repo`."
    },
    "file_text": {
      "type": "string",
      "description": "Required parameter of `create` command, with the content of the file to be created."
    },
    "insert_line": {
      "type": "integer",
      "description": "Required parameter of `insert` command. The `new_str` will be inserted AFTER the line `insert_line` of `path`."
    },
    "new_str": {
      "type": "string",
      "description": "Optional parameter of `str_replace` command containing the new string (if not given, no string will be added). Required parameter of `insert` command containing the string to insert."
    },
    "old_str": {
      "type": "string",
      "description": "Required parameter of `str_replace` command containing the string in `path` to replace."
    },
    "view_range": {
      "type": "array",
      "description": "Optional parameter of `view` command when `path` points to a file. If none is given, the full file is shown. If provided, the file will be shown in the indicated line number range, e.g. [11, 12] will show lines 11 and 12. Indexing at 1 to start. Setting `[start_line, -1]` shows all lines from `start_line` to the end of the file.",
      "items": {
        "type": "integer"
      }
    }
  },
  "required": [
    "command",
    "path"
  ]
}
```


Fuente: [`packages/fs/tool-str-replace-editor/src/index.ts`](../packages/fs/tool-str-replace-editor/src/index.ts)

Herramienta independiente de ver/crear/reemplazo literal único/inserción de líneas sobre el seam del sistema de archivos; se compone con cualquier API de shell o terminal.

<a id="deepseek-aidsh-tool-fs"></a>

## `@deepseek-ai/dsh-tool-fs`

### `edit`

Edita un archivo de texto UTF-8 existente reemplazando texto literal.

```json
{
  "type": "object",
  "properties": {
    "file_path": {
      "type": "string",
      "description": "Path to edit, resolved by the filesystem backend."
    },
    "old_string": {
      "type": "string",
      "description": "Literal text to replace. Must match exactly."
    },
    "new_string": {
      "type": "string",
      "description": "Literal replacement text. Use an empty string to delete the match."
    },
    "replace_all": {
      "type": "boolean",
      "description": "Replace all matches. Defaults to false; when false, old_string must appear exactly once."
    }
  },
  "required": [
    "file_path",
    "old_string",
    "new_string"
  ]
}
```


Fuente: [`packages/fs/tool-fs/src/index.ts`](../packages/fs/tool-fs/src/index.ts)

### `read`

Lee un archivo de texto UTF-8 y devuelve su contenido numerado por líneas.

```json
{
  "type": "object",
  "properties": {
    "file_path": {
      "type": "string",
      "description": "Path to read, resolved by the filesystem backend."
    },
    "offset": {
      "type": "number",
      "description": "1-based first line to return. Defaults to 1."
    },
    "limit": {
      "type": "number",
      "description": "Maximum number of lines to return. Defaults to 2000."
    }
  },
  "required": [
    "file_path"
  ]
}
```


Fuente: [`packages/fs/tool-fs/src/index.ts`](../packages/fs/tool-fs/src/index.ts)

### `read_image`

Lee un archivo PNG/JPEG/WebP/GIF y devuelve la imagen en sí. El harness valida y reduce las imágenes grandes admitidas antes de la siguiente solicitud al modelo, así que usa esta herramienta directamente en lugar de instalar bibliotecas de imagen o crear miniaturas solo para inspeccionar una imagen. Los archivos independientes pueden leerse de forma concurrente en lotes pequeños. Requiere que el modelo actual acepte entrada de imagen.

```json
{
  "type": "object",
  "properties": {
    "file_path": {
      "type": "string",
      "description": "Path to the image file, resolved by the filesystem backend."
    }
  },
  "required": [
    "file_path"
  ]
}
```


Fuente: [`packages/fs/tool-fs/src/index.ts`](../packages/fs/tool-fs/src/index.ts)

### `write`

Crea o reemplaza por completo un archivo de texto UTF-8.

```json
{
  "type": "object",
  "properties": {
    "file_path": {
      "type": "string",
      "description": "Path to write, resolved by the filesystem backend."
    },
    "content": {
      "type": "string",
      "description": "Full UTF-8 text content to write."
    }
  },
  "required": [
    "file_path",
    "content"
  ]
}
```


Fuente: [`packages/fs/tool-fs/src/index.ts`](../packages/fs/tool-fs/src/index.ts)

La política de leer-antes-de-escribir/editar la añade `@deepseek-ai/dsh-fs-observation-policy` (un plugin de compuerta de eventos `fs/*`, sin cambio de schema); se espera que un despliegue que carga estas herramientas también la cargue. La herramienta de imagen no se registra sin `ctx.attachments`; su schema es independiente de la ruta, y la ejecución se niega salvo que el modelo enrutado exacto declare entrada de imagen.

<a id="deepseek-aidsh-tool-fs-search"></a>

## `@deepseek-ai/dsh-tool-fs-search`

### `glob`

Encuentra archivos cuyas rutas coinciden con un patrón glob. Devuelve rutas de archivo coincidentes — nunca directorios — incluyendo archivos ocultos e ignorados (los directorios de metadatos de VCS quedan excluidos). Hasta 100 rutas vuelven en orden de hora de modificación; un resultado mayor devuelve en su lugar 100 rutas muestreadas entre las entradas de nivel superior, lo indica y notifica dónde se guardó la lista completa ordenada. Esta herramienta no enumera entradas de directorio.

```json
{
  "type": "object",
  "properties": {
    "pattern": {
      "type": "string",
      "description": "Glob pattern to match file paths against (e.g. \"**/*.ts\", \"src/**/*.test.js\"). A pattern with no \"/\" matches the basename at any depth, so \"*\" and \"*.ts\" both search the whole tree; include a separator to anchor the depth."
    },
    "path": {
      "type": "string",
      "description": "Directory to search in. Defaults to the session workspace; a relative path resolves against it."
    }
  },
  "required": [
    "pattern"
  ]
}
```


Fuente: [`packages/fs/tool-fs-search/src/index.ts`](../packages/fs/tool-fs-search/src/index.ts)

### `grep`

Busca en el contenido de archivos con una expresión regular de ripgrep. Devuelve las líneas coincidentes con sus números de línea, agrupadas por archivo. Devuelve los primeros 250 resultados en línea; un resultado con tope notifica dónde se guardó la lista completa de coincidencias. Usa read sobre un archivo coincidente para ver el contexto circundante.

```json
{
  "type": "object",
  "properties": {
    "pattern": {
      "type": "string",
      "description": "Regular expression to search for (ripgrep syntax)."
    },
    "path": {
      "type": "string",
      "description": "File or directory to search. Defaults to the session workspace; a relative path resolves against it."
    },
    "include": {
      "type": "string",
      "description": "One glob filter for which files to search (e.g. \"*.ts\", \"*.{js,jsx}\"). Not a list; negation is not supported."
    }
  },
  "required": [
    "pattern"
  ]
}
```


Fuente: [`packages/fs/tool-fs-search/src/index.ts`](../packages/fs/tool-fs-search/src/index.ts)

glob y grep son herramientas de descubrimiento incondicionales que lanzan el binario ripgrep empaquetado (`@vscode/ripgrep`) a través de ctx.subprocess como llamadas normales en primer plano (nunca trabajos en segundo plano) — sin instalación de `rg` en el host y sin capa de shell. El catálogo usa `sampleOverCapGlobResults: true`; los despliegues deben elegir ese comportamiento explícitamente. Los resultados con tope guardan la lista completa formateada a través del backend opcional ctx.spillStore; los localizadores devueltos se pueden leer/buscar después cuando el backend expone rutas locales en despliegues colocalizados.

<a id="deepseek-aidsh-tool-terminal"></a>

## `@deepseek-ai/dsh-tool-terminal`

### `terminal_close`

Cierra un terminal persistente y espera a que desaparezca su árbol de procesos propios capturado.

```json
{
  "type": "object",
  "properties": {
    "sessionId": {
      "type": "string",
      "description": "Terminal session id."
    }
  },
  "required": [
    "sessionId"
  ]
}
```


Fuente: [`packages/terminal/tool-terminal/src/index.ts`](../packages/terminal/tool-terminal/src/index.ts)

### `terminal_list`

Lista las sesiones de terminal persistente propiedad del agent actual.

```json
{
  "type": "object",
  "properties": {}
}
```


Fuente: [`packages/terminal/tool-terminal/src/index.ts`](../packages/terminal/tool-terminal/src/index.ts)

### `terminal_open`

Crea una sesión de terminal persistente y aislada por propietario a partir de un tipo de backend registrado. Úsala para el estado de shell o REPL que deba sobrevivir entre llamadas de herramienta.

```json
{
  "type": "object",
  "properties": {
    "type": {
      "type": "string",
      "description": "Registered terminal backend type, usually \"shell\"."
    },
    "name": {
      "type": "string",
      "description": "Optional owner-local display name such as \"main\" or \"gdb\"."
    },
    "cwd": {
      "type": "string",
      "description": "Initial working directory. Defaults to the deployment workspace root."
    }
  },
  "required": [
    "type"
  ]
}
```


Fuente: [`packages/terminal/tool-terminal/src/index.ts`](../packages/terminal/tool-terminal/src/index.ts)

### `terminal_read`

Lee una página acotada de la salida retenida de un terminal persistente sin enviar entrada.

```json
{
  "type": "object",
  "properties": {
    "sessionId": {
      "type": "string",
      "description": "Terminal session id."
    },
    "offset": {
      "type": "number",
      "description": "Newest-relative line offset (default 0)."
    },
    "count": {
      "type": "number",
      "description": "Requested line count (default 500; backend caps apply)."
    }
  },
  "required": [
    "sessionId"
  ]
}
```


Fuente: [`packages/terminal/tool-terminal/src/index.ts`](../packages/terminal/tool-terminal/src/index.ts)

### `terminal_send`

Envía texto a un terminal persistente. Por defecto se envía Enter y la llamada espera un prompt, una espera de stdin, silencio de salida, timeout o la salida de la sesión. El modo en segundo plano devuelve un id de trabajo para job_output/job_kill.

```json
{
  "type": "object",
  "properties": {
    "sessionId": {
      "type": "string",
      "description": "Terminal session id returned by terminal_open or terminal_list."
    },
    "text": {
      "type": "string",
      "description": "UTF-8 text to write to the terminal."
    },
    "submit": {
      "type": "boolean",
      "description": "Submit Enter after text (default true). Set false for control characters or incomplete REPL input."
    },
    "run_in_background": {
      "type": "boolean",
      "description": "Return a job id immediately; collect with job_output or stop with job_kill."
    }
  },
  "required": [
    "sessionId",
    "text"
  ]
}
```


Fuente: [`packages/terminal/tool-terminal/src/index.ts`](../packages/terminal/tool-terminal/src/index.ts)

### `terminal_signal`

Envía una señal permitida al grupo de procesos en primer plano actual de un terminal persistente.

```json
{
  "type": "object",
  "properties": {
    "sessionId": {
      "type": "string",
      "description": "Terminal session id."
    },
    "signal": {
      "type": "string",
      "description": "Signal to deliver. Shell-targeted SIGKILL is rejected; use terminal_close.",
      "enum": [
        "SIGINT",
        "SIGTERM",
        "SIGKILL",
        "SIGTSTP",
        "SIGHUP"
      ]
    }
  },
  "required": [
    "sessionId",
    "signal"
  ]
}
```


Fuente: [`packages/terminal/tool-terminal/src/index.ts`](../packages/terminal/tool-terminal/src/index.ts)

Las seis herramientas de terminal son opt-in y complementan las herramientas de shell/sistema de archivos de un solo uso. `terminal_send(run_in_background: true)` se registra en `ctx.jobs`; TUI, secuencias de teclas con nombre, BEL, redimensionado, auto-inicio y compartición entre agents están ausentes del schema.

<a id="deepseek-aidsh-tool-goal"></a>

## `@deepseek-ai/dsh-tool-goal`

### `create_goal`

Crea un objetivo de finalización persistente de la misma sesión cuando la solicitud humana directa actual es un objetivo de larga duración que debería continuar a través de rondas de objetivo autónomas. Puedes inferir esa intención sin exigir que el usuario diga "create a goal". No lo uses para trabajo trivial de un solo turno. La ejecución rechaza la autoridad no humana y la de subagentes.

```json
{
  "type": "object",
  "properties": {
    "objective": {
      "type": "string",
      "description": "The concrete completion objective inferred from the direct human request."
    },
    "max_goal_rounds": {
      "type": "number",
      "description": "Optional positive safe-integer limit on automatic continuation rounds."
    }
  },
  "required": [
    "objective"
  ]
}
```


Fuente: [`packages/goal/tool-goal/src/index.ts`](../packages/goal/tool-goal/src/index.ts)

### `get_goal`

Lee el objetivo actual de la misma sesión, incluidos su id/revisión exactos, el objetivo, la fase, las rondas de continuación completadas, el límite de rondas, el motivo del bloqueo cuando exista y si hay otra continuación armada. Llama a esto antes de actualizar un objetivo.

```json
{
  "type": "object",
  "properties": {}
}
```


Fuente: [`packages/goal/tool-goal/src/index.ts`](../packages/goal/tool-goal/src/index.ts)

### `update_goal`

Actualiza la revisión exacta del objetivo actual. edit, pause y resume requieren una solicitud humana directa de nivel superior. Durante una continuación automática del objetivo actual, complete y blocked también están permitidos. blocked se rechaza antes del número mínimo de rondas configurado; el modelo sigue siendo responsable de juzgar que la misma condición persistió a lo largo de esas rondas y debe explicarlo en blocked_reason.

```json
{
  "type": "object",
  "properties": {
    "goal_id": {
      "type": "string",
      "description": "Exact id returned by get_goal."
    },
    "revision": {
      "type": "number",
      "description": "Exact positive revision returned by get_goal."
    },
    "action": {
      "type": "string",
      "description": "edit | pause | resume | complete | blocked",
      "enum": [
        "edit",
        "pause",
        "resume",
        "complete",
        "blocked"
      ]
    },
    "objective": {
      "type": "string",
      "description": "Replacement objective; valid only with action edit."
    },
    "max_goal_rounds": {
      "type": "number",
      "description": "Replacement cap; valid only with action edit."
    },
    "blocked_reason": {
      "type": "string",
      "description": "Concrete blocking condition; required only with action blocked."
    }
  },
  "required": [
    "goal_id",
    "revision",
    "action"
  ]
}
```


Fuente: [`packages/goal/tool-goal/src/index.ts`](../packages/goal/tool-goal/src/index.ts)

create, edit, pause y resume requieren autoridad raíz humana directa; complete y blocked también aceptan la ronda actual exacta del objetivo. El límite inferior por defecto de blocked es de tres rondas admitidas.

<a id="deepseek-aidsh-schedule"></a>

## `@deepseek-ai/dsh-schedule`

### `schedule_create`

Crea un recordatorio en la sesión actual. Proporciona un prompt no vacío y exactamente un selector: un retraso after_seconds de entero seguro positivo, at como fecha-hora con offset estricto u objeto de fecha/hora local, o every_seconds de entero seguro de al menos 300. Los recordatorios de tasa fija se alinean con su creación, se saltan las ocurrencias perdidas y agrupan una única ocurrencia más reciente por regla vencida. La entrega es local a la sesión: el recordatorio se ejecuta a tiempo solo mientras esta sesión está viva y, de lo contrario, queda vencido hasta que la sesión se reanuda.

```json
{
  "type": "object",
  "properties": {
    "prompt": {
      "type": "string",
      "description": "Reminder content to present when the target becomes due."
    },
    "after_seconds": {
      "type": "number",
      "description": "Positive safe-integer delay in seconds."
    },
    "every_seconds": {
      "type": "number",
      "description": "Fixed-rate safe-integer interval in seconds, at least 300."
    },
    "at": {
      "oneOf": [
        {
          "type": "string"
        },
        {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "date": {
              "type": "string"
            },
            "time": {
              "type": "string"
            },
            "time_zone": {
              "type": "string"
            }
          },
          "required": [
            "date",
            "time",
            "time_zone"
          ]
        }
      ],
      "description": "Absolute target as strict offset RFC 3339 or local date/time with an explicit IANA zone."
    }
  },
  "required": [
    "prompt"
  ]
}
```


Fuente: [`packages/schedule/schedule/src/tools.ts`](../packages/schedule/schedule/src/tools.ts)

### `schedule_delete`

Elimina un recordatorio activo de la sesión actual por el id exacto devuelto por schedule_create o schedule_list. Los ids desconocidos o ya finalizados devuelven deleted false.

```json
{
  "type": "object",
  "properties": {
    "id": {
      "type": "string",
      "description": "Exact session-local schedule id."
    }
  },
  "required": [
    "id"
  ]
}
```


Fuente: [`packages/schedule/schedule/src/tools.ts`](../packages/schedule/schedule/src/tools.ts)

### `schedule_list`

Lista cada recordatorio activo de la sesión actual en orden de creación, incluidos su id exacto, el objetivo UTC, el estado programado o vencido y el modo de entrega local a la sesión.

```json
{
  "type": "object",
  "properties": {}
}
```


Fuente: [`packages/schedule/schedule/src/tools.ts`](../packages/schedule/schedule/src/tools.ts)

Solo se registra dentro de ámbitos de Agent raíz en vivo creados después de que cargue el plugin de Schedule opt-in. La versión 1 acepta after_seconds, at absoluto explícito y every_seconds de tasa fija acotado, y divulga la entrega local a la sesión; las lecturas y mutaciones de gestión requieren la barrera compartida de persistencia de sesión.

<a id="deepseek-aidsh-tool-lsp"></a>

## `@deepseek-ai/dsh-tool-lsp`

### `lsp`

Consulta un language server para una navegación de código precisa. operation es uno de goToDefinition, findReferences, goToImplementation, hover. line y character son coordenadas de cursor UTF-16 basadas en uno. findReferences incluye la declaración.

```json
{
  "type": "object",
  "properties": {
    "operation": {
      "type": "string",
      "description": "goToDefinition, findReferences, goToImplementation, or hover.",
      "enum": [
        "goToDefinition",
        "findReferences",
        "goToImplementation",
        "hover"
      ]
    },
    "file_path": {
      "type": "string",
      "description": "The source file to query, relative to the workspace or absolute."
    },
    "line": {
      "type": "number",
      "description": "One-based line of the cursor."
    },
    "character": {
      "type": "number",
      "description": "One-based UTF-16 column of the cursor."
    }
  },
  "required": [
    "operation",
    "file_path",
    "line",
    "character"
  ]
}
```


Fuente: [`packages/lsp/tool-lsp/src/index.ts`](../packages/lsp/tool-lsp/src/index.ts)

La herramienta lsp mantiene la selección de provider y los subprocesos del language server detrás de ctx.lsp, por lo que su schema visible para el modelo permanece estable entre providers. Requiere un provider registrado (p. ej. `@deepseek-ai/dsh-lsp-stdio`) en tiempo de ejecución; sin uno, una consulta devuelve el error estructurado `LSP_UNAVAILABLE` en lugar de cambiar el schema.

<a id="deepseek-aidsh-tool-ralph"></a>

## `@deepseek-ai/dsh-tool-ralph`

### `ralph`

Ejecuta un bucle Ralph de agent nuevo en primer plano hacia un objetivo inmutable. Úsalo solo cuando el humano directo pide explícitamente Ralph o iteración de agent nuevo. Cada ronda abre un hijo nuevo sin conversación padre ni sesión de hijo anterior; el workspace compartido es la memoria a largo plazo, y solo un informe estructurado acotado cruza las rondas. La llamada devuelve cuando un worker notifica finalización o un bloqueo concreto, o en el límite de rondas. El trabajo ordinario de larga duración en la misma sesión pertenece a las herramientas de objetivo.

```json
{
  "type": "object",
  "properties": {
    "objective": {
      "type": "string",
      "description": "The immutable completion objective for every fresh Ralph round."
    },
    "maxRounds": {
      "type": "number",
      "description": "Optional positive safe-integer round cap, bounded by the deployment ceiling."
    }
  },
  "required": [
    "objective"
  ]
}
```


Fuente: [`packages/workflow/tool-ralph/src/index.ts`](../packages/workflow/tool-ralph/src/index.ts)

Un flujo de trabajo fijo en primer plano inicia un hijo estructurado nuevo por ronda; el modelo solo selecciona el objetivo inmutable y un tope de rondas opcional.

<a id="deepseek-aidsh-tool-skill"></a>

## `@deepseek-ai/dsh-tool-skill`

### `skill`

Carga las instrucciones completas de una skill disponible. Llama a esto con el nombre exacto de la skill del catálogo de skills de la sesión antes de actuar en una tarea que nombre esa skill o coincida claramente con ella.

```json
{
  "type": "object",
  "properties": {
    "name": {
      "type": "string",
      "description": "The exact skill name from the available skills list."
    }
  },
  "required": [
    "name"
  ]
}
```


Fuente: [`packages/skill/tool-skill/src/index.ts`](../packages/skill/tool-skill/src/index.ts)

<a id="deepseek-aidsh-tool-session-query"></a>

## `@deepseek-ai/dsh-tool-session-query`

### `session_event_read`

Lee un evento completo sin abreviar y los resúmenes opcionales de eventos en bruto vecinos de una sesión autorizada.

```json
{
  "type": "object",
  "properties": {
    "session_id": {
      "type": "string",
      "description": "Target session id. Omit for the current session."
    },
    "seq": {
      "type": "integer",
      "description": "Target event sequence number."
    },
    "before": {
      "type": "integer",
      "description": "Number of preceding raw events to summarize. Omit for none."
    },
    "after": {
      "type": "integer",
      "description": "Number of following raw events to summarize. Omit for none."
    }
  },
  "required": [
    "seq"
  ]
}
```


Fuente: [`packages/session-query/tool-session-query/src/index.ts`](../packages/session-query/tool-session-query/src/index.ts)

### `session_event_search`

Busca eventos anteriores en una sesión autorizada; la sesión actual excluye el paso que realiza esta llamada.

```json
{
  "type": "object",
  "properties": {
    "session_id": {
      "type": "string",
      "description": "Target session id. Omit for the current session."
    },
    "query": {
      "type": "string",
      "description": "Literal full-text query over the target session."
    },
    "seq_from": {
      "type": "integer",
      "description": "Inclusive event sequence lower bound."
    },
    "seq_to": {
      "type": "integer",
      "description": "Inclusive event sequence upper bound."
    },
    "time_from": {
      "type": "string",
      "description": "Inclusive timezone-qualified ISO 8601 event-time lower bound."
    },
    "time_to": {
      "type": "string",
      "description": "Inclusive timezone-qualified ISO 8601 event-time upper bound."
    },
    "event_types": {
      "type": "array",
      "description": "Event types to include.",
      "items": {
        "type": "string"
      }
    },
    "surfaces": {
      "type": "array",
      "description": "Event surfaces to include.",
      "items": {
        "type": "string",
        "enum": [
          "current",
          "shadowed",
          "log-only"
        ]
      }
    }
  },
  "required": [
    "query"
  ]
}
```


Fuente: [`packages/session-query/tool-session-query/src/index.ts`](../packages/session-query/tool-session-query/src/index.ts)

### `session_event_trace`

Lee cada reemplazo directo y la relación con un evento de origen citado para un evento de una sesión autorizada.

```json
{
  "type": "object",
  "properties": {
    "session_id": {
      "type": "string",
      "description": "Target session id. Omit for the current session."
    },
    "seq": {
      "type": "integer",
      "description": "Target event sequence number."
    }
  },
  "required": [
    "seq"
  ]
}
```


Fuente: [`packages/session-query/tool-session-query/src/index.ts`](../packages/session-query/tool-session-query/src/index.ts)

### `session_search`

Busca sesiones anteriores en el workspace del llamador y devuelve el evento de mayor coincidencia de cada sesión.

```json
{
  "type": "object",
  "properties": {
    "query": {
      "type": "string",
      "description": "Literal full-text query over prior session history."
    },
    "session_ids": {
      "type": "array",
      "description": "Optional session ids to include.",
      "items": {
        "type": "string"
      }
    },
    "created_at_from": {
      "type": "string",
      "description": "Inclusive timezone-qualified ISO 8601 creation-time lower bound."
    },
    "created_at_to": {
      "type": "string",
      "description": "Inclusive timezone-qualified ISO 8601 creation-time upper bound."
    },
    "parent_session_ids": {
      "type": "array",
      "description": "Optional direct parent session ids.",
      "items": {
        "type": "string"
      }
    },
    "include_root_sessions": {
      "type": "boolean",
      "description": "Include sessions with no parent in the parent filter."
    },
    "availability": {
      "type": "array",
      "description": "Require at least one selected source availability.",
      "items": {
        "type": "string",
        "enum": [
          "live",
          "persisted"
        ]
      }
    },
    "event_seq_from": {
      "type": "integer",
      "description": "Inclusive event sequence lower bound."
    },
    "event_seq_to": {
      "type": "integer",
      "description": "Inclusive event sequence upper bound."
    },
    "event_time_from": {
      "type": "string",
      "description": "Inclusive timezone-qualified ISO 8601 event-time lower bound."
    },
    "event_time_to": {
      "type": "string",
      "description": "Inclusive timezone-qualified ISO 8601 event-time upper bound."
    },
    "event_types": {
      "type": "array",
      "description": "Event types to include.",
      "items": {
        "type": "string"
      }
    },
    "event_surfaces": {
      "type": "array",
      "description": "Event surfaces to include.",
      "items": {
        "type": "string",
        "enum": [
          "current",
          "shadowed",
          "log-only"
        ]
      }
    }
  },
  "required": [
    "query"
  ]
}
```


Fuente: [`packages/session-query/tool-session-query/src/index.ts`](../packages/session-query/tool-session-query/src/index.ts)

### `session_trace`

Lee el linaje de sesiones autorizado alrededor de una sesión, incluidas las relaciones completas visibles de ancestros y descendientes.

```json
{
  "type": "object",
  "properties": {
    "session_id": {
      "type": "string",
      "description": "Target session id. Omit for the current session."
    }
  }
}
```


Fuente: [`packages/session-query/tool-session-query/src/index.ts`](../packages/session-query/tool-session-query/src/index.ts)

Las cinco herramientas de solo lectura ocultan los cursores de los providers y autorizan cada resultado desde la sesión inmutable del agent que llama. El paquete es opt-in; las composiciones que necesitan plazos exigidos o salida en línea acotada también montan las políticas genéricas de timeout o spill.

<a id="deepseek-aidsh-tool-subagent"></a>

## `@deepseek-ai/dsh-tool-subagent`

### `subagent`

Delega una tarea autocontenida a un subagente (un agent separado que trabaja en su propio contexto) para descargar trabajo enfocado e independiente — investigación, una implementación acotada, un análisis — de modo que no consuma el contexto de esta conversación. El subagente devuelve su resultado, no sus pasos intermedios. Dale un prompt completo e independiente: no ve esta conversación. Esta llamada espera el resultado por defecto. Pon `run_in_background: true` para devolver un id de trabajo; recógelo con `job_output` y detenlo con `job_kill`.

```json
{
  "type": "object",
  "properties": {
    "description": {
      "type": "string",
      "description": "A short (3-5 word) description of the delegated task, for display."
    },
    "prompt": {
      "type": "string",
      "description": "The complete, self-contained task for the subagent. It does not share this conversation's context, so include everything it needs."
    },
    "run_in_background": {
      "type": "boolean",
      "description": "Whether to run as a background job and return its id. Defaults to false; collect with job_output or stop with job_kill."
    }
  },
  "required": [
    "description",
    "prompt"
  ]
}
```


Fuente: [`packages/subagent/tool-subagent/src/index.ts`](../packages/subagent/tool-subagent/src/index.ts)

El nombre de herramienta registrado es la configuración de tiempo de carga `toolName` (por defecto `subagent`); el schema anterior es ese valor por defecto. Las composiciones incluidas cargan este paquete una vez por backend de subagente, por lo que el modelo además ve `subagent_fork` enlazado al backend de fork. La descripción de cada instancia, el parámetro `run_in_background` y la política de system prompt siguen su propio `backgroundMode` y `enableRunInBackground`, así que los dos schemas incluidos no son idénticos: `subagent` es `continuable` y por defecto envía las llamadas omitidas al fondo con entrega automática de liquidación, mientras que `subagent_fork` sigue siendo `one-shot` y por defecto las envía al primer plano — consulta `packages/bundle/base/cordis.patch.yml` y `examples/acp-agent/cordis.yml`.

<a id="deepseek-aidsh-tool-subagent-control"></a>

## `@deepseek-ai/dsh-tool-subagent-control`

### `interrupt_agent`

Solicita la cancelación del turno actual de un agent en segundo plano por su agent id. El objetivo puede ser tu hijo directo o un agent más profundo creado bajo ti. Solo se detiene el turno actual: los mensajes ya encolados para el agent permanecen aparcados hasta un send_message posterior, los agents que inició siguen ejecutándose y el propio agent sigue disponible para seguimientos. Esta llamada devuelve en cuanto se acepta la solicitud de detención, así que el objetivo puede seguir ejecutándose brevemente; interrumpir un agent que ya terminó es un no-op aceptado.

```json
{
  "type": "object",
  "properties": {
    "agent_id": {
      "type": "string",
      "description": "The agent id of the running agent to interrupt."
    }
  },
  "required": [
    "agent_id"
  ]
}
```


Fuente: [`packages/subagent/tool-subagent-control/src/index.ts`](../packages/subagent/tool-subagent-control/src/index.ts)

### `list_agents`

Lista tus subagentes de fondo continuables por id durable y etiqueta. Úsalo para recordar cuáles iniciaste, no para sondear la finalización — te avisan cuando uno termina. El estado proviene del registro en vivo: running significa que el agent está trabajando ahora mismo, idle significa que está cargado pero entre turnos (puede estar esperando a agents que inició), y ready significa que solo existe en el almacenamiento — reanudable, no terminal, y no un resultado esperando a recogerse; un `send_message` inicia un turno nuevo en la misma conversación, y un hijo directo sigue siendo candidato de `send_message` en cualquier estado. La instantánea no es una promesa de entrega — `send_message` realiza la comprobación autoritativa y puede fallar igualmente. Los hijos que no pudieron leerse se notifican como diagnósticos en lugar de descartarse en silencio. El ámbito `descendants` recorre todo el árbol bajo ti en preorden estable, anotando cada entrada con su id de sesión de padre directo durable y su profundidad. Puedes usar `send_message` solo para entradas de profundidad 1; las más profundas son candidatas solo de `interrupt_agent`.

```json
{
  "type": "object",
  "properties": {
    "scope": {
      "type": "string",
      "description": "children (default) lists direct children only; descendants walks the complete tree below you.",
      "enum": [
        "children",
        "descendants"
      ]
    }
  }
}
```


Fuente: [`packages/subagent/tool-subagent-control/src/list-agents.ts`](../packages/subagent/tool-subagent-control/src/list-agents.ts)

### `send_message`

Envía un mensaje a un subagente de fondo por su id de subagente, continuando la misma conversación. Se convierte en el siguiente turno del subagente: si todavía está trabajando, el mensaje espera hasta que termina su turno actual, así que no puede redirigir el trabajo ya en marcha. Esta llamada no devuelve respuesta del subagente — solo la confirmación de que el mensaje se entregó —, así que úsala para darle más trabajo. Un fallo significa que el mensaje NO se entregó.

```json
{
  "type": "object",
  "properties": {
    "subagent_id": {
      "type": "string",
      "description": "The subagent id returned when the background subagent was started."
    },
    "message": {
      "type": "string",
      "description": "The message to deliver to the subagent."
    }
  },
  "required": [
    "subagent_id",
    "message"
  ]
}
```


Fuente: [`packages/subagent/tool-subagent-control/src/index.ts`](../packages/subagent/tool-subagent-control/src/index.ts)

Las herramientas de control con nombre global sobre subagentes de fondo continuables: las instancias de `tool-subagent` enlazadas a un provider registran herramientas de delegación distintas, mientras que este paquete registra `send_message` e `interrupt_agent` una sola vez, además de `list_agents` desde su plugin `/list-agents` cargado por separado (cuyas filas del catálogo usan los registros sessionProjections y de Agent en vivo).

<a id="deepseek-aidsh-tool-subagent-report"></a>

## `@deepseek-ai/dsh-tool-subagent-report`

### `report`

Notifica contenido seleccionado al agent que te inició. Llama a esto una vez antes de terminar, con un resultado final autocontenido, y antes para progresos o hallazgos que cambien lo que ese agent haga después. Ese agent comparte tu workspace pero no recibe automáticamente tu transcript (transcripción), la salida de tus herramientas ni tu razonamiento, así que terminar tu trabajo no es en sí un resultado. Notificar no termina tu turno ni acaba tu trabajo, y solo tu padre directo lo recibe. Una llamada fallida puede haber llegado igualmente, así que no la repitas a ciegas.

```json
{
  "type": "object",
  "properties": {
    "output": {
      "type": "string",
      "description": "Actionable content for your parent; summarize conclusions and reference relevant shared paths."
    }
  },
  "required": [
    "output"
  ]
}
```


Fuente: [`packages/subagent/tool-subagent-report/src/index.ts`](../packages/subagent/tool-subagent-report/src/index.ts)

Se registra por hijo continuable en proceso en lugar de globalmente, por lo que este schema solo es visible dentro de ese hijo y sobrevive a su `toolFilter` global. La misma contribución instala la sección de prompt `tool:report` con ámbito de hijo, que este catálogo no renderiza. La herramienta `send_message` orientada al padre se instala de forma independiente.

<a id="deepseek-aidsh-tool-jobs"></a>

## `@deepseek-ai/dsh-tool-jobs`

### `job_kill`

Solicita la cancelación de un trabajo en segundo plano en ejecución por su id de trabajo. Devuelve de inmediato; el trabajo se liquida como killed una vez que su trabajo se detiene de verdad.

```json
{
  "type": "object",
  "properties": {
    "job_id": {
      "type": "string",
      "description": "Job id returned by the tool that started the background work."
    },
    "reason": {
      "type": "string",
      "description": "Optional short reason, recorded in the log and forwarded to the job."
    }
  },
  "required": [
    "job_id"
  ]
}
```


Fuente: [`packages/jobs/tool-jobs/src/index.ts`](../packages/jobs/tool-jobs/src/index.ts)

### `job_list`

Lista tus trabajos en segundo plano (en ejecución y finalizados) con sus ids, tipos y estados.

```json
{
  "type": "object",
  "properties": {}
}
```


Fuente: [`packages/jobs/tool-jobs/src/index.ts`](../packages/jobs/tool-jobs/src/index.ts)

### `job_output`

Lee un trabajo en segundo plano. Los trabajos de stream devuelven solo la salida desde la lectura anterior; los trabajos de salida final devuelven su resultado tras la liquidación. Cada respuesta termina con `[status: ...]`. Las lecturas no bloquean salvo con `wait: true`, que espera hasta el tope configurado.

```json
{
  "type": "object",
  "properties": {
    "job_id": {
      "type": "string",
      "description": "Job id returned by the tool that started the background work."
    },
    "wait": {
      "type": "boolean",
      "description": "Block until the job reaches a terminal status or the timeout expires. A timed-out wait returns [status: running] and leaves the job alive."
    },
    "timeout_ms": {
      "type": "number",
      "description": "Max wait in milliseconds (only meaningful with wait: true). Defaults to the configured wait timeout; capped by the configured maximum."
    }
  },
  "required": [
    "job_id"
  ]
}
```


Fuente: [`packages/jobs/tool-jobs/src/index.ts`](../packages/jobs/tool-jobs/src/index.ts)

El controlador de trabajos en segundo plano agnóstico al tipo: los comandos bash en segundo plano, los envíos PTY y los subagentes se leen, listan y matan con las mismas tres herramientas. Cargar el plugin adjunta el controlador que arma el `ctx.jobs.start()` de los productores.

<a id="deepseek-aidsh-experimental-tool-agent-team"></a>

## `@deepseek-ai/dsh-experimental-tool-agent-team`

### `followup_task`

Envía una tarea de seguimiento durable a otro miembro del Team e inicia un turno cuando haga falta.

```json
{
  "type": "object",
  "properties": {
    "target": {
      "type": "string",
      "description": "Team member name, or lead."
    },
    "message": {
      "type": "string",
      "description": "Self-contained message for the target."
    }
  },
  "required": [
    "target",
    "message"
  ]
}
```


Fuente: [`packages/experimental/tool-agent-team/src/index.ts`](../packages/experimental/tool-agent-team/src/index.ts)

### `interrupt_agent`

Interrumpe el turno actual de un compañero de equipo conservando su bandeja de entrada pendiente. Solo Team Lead.

```json
{
  "type": "object",
  "properties": {
    "target": {
      "type": "string",
      "description": "Teammate name."
    }
  },
  "required": [
    "target"
  ]
}
```


Fuente: [`packages/experimental/tool-agent-team/src/index.ts`](../packages/experimental/tool-agent-team/src/index.ts)

### `list_agents`

Lista el Lead y cada compañero de equipo durable con su estado de runtime actual.

```json
{
  "type": "object",
  "properties": {}
}
```


Fuente: [`packages/experimental/tool-agent-team/src/index.ts`](../packages/experimental/tool-agent-team/src/index.ts)

### `send_message`

Envía información durable a otro miembro del Team sin iniciar a un miembro inactivo.

```json
{
  "type": "object",
  "properties": {
    "target": {
      "type": "string",
      "description": "Team member name, or lead."
    },
    "message": {
      "type": "string",
      "description": "Self-contained message for the target."
    }
  },
  "required": [
    "target",
    "message"
  ]
}
```


Fuente: [`packages/experimental/tool-agent-team/src/index.ts`](../packages/experimental/tool-agent-team/src/index.ts)

### `spawn_teammate`

Crea un compañero de equipo con nombre y durable. Solo el Team Lead puede llamar a esta herramienta.

```json
{
  "type": "object",
  "properties": {
    "name": {
      "type": "string",
      "description": "Unique lower-kebab-case teammate name."
    },
    "description": {
      "type": "string",
      "description": "Short description of the delegated responsibility."
    },
    "prompt": {
      "type": "string",
      "description": "Complete initial task for the teammate."
    },
    "context": {
      "type": "string",
      "description": "fresh starts without Lead history; fork inherits completed Lead turns. Defaults to fresh.",
      "enum": [
        "fresh",
        "fork"
      ]
    }
  },
  "required": [
    "name",
    "description",
    "prompt"
  ]
}
```


Fuente: [`packages/experimental/tool-agent-team/src/index.ts`](../packages/experimental/tool-agent-team/src/index.ts)

### `team_task_create`

Crea una tarea pendiente sin propietario en el tablero de tareas compartido del Team.

```json
{
  "type": "object",
  "properties": {
    "subject": {
      "type": "string",
      "description": "Concise task title."
    },
    "description": {
      "type": "string",
      "description": "Complete task details and acceptance criteria."
    },
    "blocked_by": {
      "type": "array",
      "description": "Task ids that must complete first.",
      "items": {
        "type": "string"
      }
    },
    "write_scopes": {
      "type": "array",
      "description": "Advisory workspace-relative file or directory prefixes this task expects to modify.",
      "items": {
        "type": "string"
      }
    }
  },
  "required": [
    "subject",
    "description"
  ]
}
```


Fuente: [`packages/experimental/tool-agent-team/src/index.ts`](../packages/experimental/tool-agent-team/src/index.ts)

### `team_task_get`

Lee el valor completo más reciente de una tarea compartida antes de cambiarla o ejecutarla.

```json
{
  "type": "object",
  "properties": {
    "task_id": {
      "type": "string",
      "description": "Shared task id."
    }
  },
  "required": [
    "task_id"
  ]
}
```


Fuente: [`packages/experimental/tool-agent-team/src/index.ts`](../packages/experimental/tool-agent-team/src/index.ts)

### `team_task_list`

Lista las tareas compartidas, incluidos disponibilidad, propietario, revisión, bloqueadores y avisos de ámbito de escritura.

```json
{
  "type": "object",
  "properties": {
    "status": {
      "type": "string",
      "description": "Optional exact status filter.",
      "enum": [
        "pending",
        "in_progress",
        "completed"
      ]
    },
    "owner": {
      "type": "string",
      "description": "Optional member-name filter; use unowned for tasks without an owner."
    },
    "ready": {
      "type": "boolean",
      "description": "Optional readiness filter."
    },
    "cursor": {
      "type": "integer",
      "description": "Zero-based result offset. Defaults to 0."
    },
    "limit": {
      "type": "integer",
      "description": "Number of rows, 1 through 100. Defaults to 50."
    }
  }
}
```


Fuente: [`packages/experimental/tool-agent-team/src/index.ts`](../packages/experimental/tool-agent-team/src/index.ts)

### `team_task_update`

Aplica una acción de tarea compartida con compare-and-set usando la revisión más reciente de team_task_get o team_task_list.

```json
{
  "type": "object",
  "properties": {
    "task_id": {
      "type": "string",
      "description": "Shared task id."
    },
    "expected_revision": {
      "type": "integer",
      "description": "Current task revision used as the CAS precondition."
    },
    "action": {
      "type": "string",
      "description": "Task transition to apply.",
      "enum": [
        "claim",
        "release",
        "edit",
        "set_dependencies",
        "complete",
        "reopen",
        "reassign",
        "delete"
      ]
    },
    "subject": {
      "type": "string",
      "description": "Replacement title for edit."
    },
    "description": {
      "type": "string",
      "description": "Replacement details for edit."
    },
    "blocked_by": {
      "type": "array",
      "description": "Complete blocker list for set_dependencies.",
      "items": {
        "type": "string"
      }
    },
    "write_scopes": {
      "type": "array",
      "description": "Replacement advisory write scopes for edit.",
      "items": {
        "type": "string"
      }
    },
    "owner": {
      "type": "string",
      "description": "Member name for Lead-only reassign; omit to unassign."
    }
  },
  "required": [
    "task_id",
    "expected_revision",
    "action"
  ]
}
```


Fuente: [`packages/experimental/tool-agent-team/src/index.ts`](../packages/experimental/tool-agent-team/src/index.ts)

### `wait_agent`

Espera el siguiente cambio de estado de compañero, buzón o tarea compartida después de que esta llamada comience. Esto nunca despierta a miembros inactivos y devuelve noProgress de inmediato cuando ningún otro miembro está en ejecución o aprovisionándose. Vuelve a listar tras el despertar o el timeout en lugar de sondear.

```json
{
  "type": "object",
  "properties": {
    "timeout_ms": {
      "type": "integer",
      "description": "Wait duration in milliseconds, from 10000 through 3600000. Defaults to 30000."
    }
  }
}
```


Fuente: [`packages/experimental/tool-agent-team/src/index.ts`](../packages/experimental/tool-agent-team/src/index.ts)

Las diez herramientas tienen ámbito para Team Leads implícitos y compañeros de equipo durables. El bundle dsh-base incluido mantiene el paquete desactivado; el parche de perfil Agent Teams documentado lo activa mientras desactiva los nombres de control heredados de hijos continuables.

<a id="deepseek-aidsh-tool-todo"></a>

## `@deepseek-ai/dsh-tool-todo`

### `todo_write`

Registra y actualiza una lista de tareas estructurada para el trabajo actual. Envía la lista ENTERA en cada llamada — REEMPLAZA la lista anterior (no hay actualizaciones parciales ni ediciones por elemento). Úsala para planificar trabajo de varios pasos y mostrar progreso: añade un todo por paso concreto antes de empezar. Marca cada todo en el que estés trabajando activamente como `in_progress` — varios a la vez cuando el trabajo corre de verdad en paralelo (p. ej. subagentes concurrentes o comandos en segundo plano), uno para trabajo secuencial; mientras quede trabajo, al menos una tarea debería estar `in_progress`. Marca un todo como `completed` en el momento en que termina (no agrupes finalizaciones), y no permitas ningún elemento `in_progress` solo cuando todo el trabajo esté completo. Omite la lista para tareas triviales de un solo paso. Estados: `pending` (no iniciado), `in_progress` (en trabajo ahora), `completed` (finalizado).

```json
{
  "type": "object",
  "properties": {
    "todos": {
      "type": "array",
      "description": "The COMPLETE task list, replacing any previous list.",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "content": {
            "type": "string",
            "description": "What the task is — a short imperative line."
          },
          "status": {
            "type": "string",
            "description": "pending (not started) | in_progress (now) | completed (done).",
            "enum": [
              "pending",
              "in_progress",
              "completed"
            ]
          }
        },
        "required": [
          "content",
          "status"
        ]
      }
    }
  },
  "required": [
    "todos"
  ]
}
```


Fuente: [`packages/todo/tool-todo/src/index.ts`](../packages/todo/tool-todo/src/index.ts)

todo_write es estado propiedad de la sesión; las UIs renderizan el último evento todo/write como una lista de verificación. `allowParallelInProgress` es obligatorio y no tiene valor por defecto, así que el catálogo declara su elección: `true`, cuya descripción invita a varios elementos `in_progress`. Un despliegue que elige `false` recibe la misma herramienta con una descripción que pide exactamente una tarea activa.

<a id="deepseek-aidsh-tool-workflow"></a>

## `@deepseek-ai/dsh-tool-workflow`

### `workflow`

Ejecuta un script de flujo de trabajo JavaScript que orquesta subagentes a escala. Úsalo para trabajo que se ramifica en muchas piezas independientes — una auditoría sobre muchos archivos, una migración, investigación desde varios ángulos, verificación adversarial de hallazgos — donde escribes la orquestación como un script en lugar de delegar turno a turno.

La identidad del flujo de trabajo viaja en el parámetro `meta` como JSON: los strings obligatorios `name` (kebab-case corto) y `description`, y el string opcional `whenToUse` y el array `phases` (`{title, detail?, provider?, model?}`). El parámetro `script` es SOLO el cuerpo de JavaScript plano (NO TypeScript, y NINGUNA declaración `export const meta` — meta es un parámetro, no código), que se ejecuta con await de nivel superior; termina con `return <value>` — el valor debe ser serializable a JSON y es el resultado de esta herramienta.

Hooks del cuerpo del script:
- `agent(prompt, opts?): Promise<any>` — ejecuta un subagente hasta que termina. Sin `opts.schema` se resuelve al texto final del hijo; con `opts.schema` (un JSON Schema con raíz de objeto que usa SOLO type/properties/required/additionalProperties/items/enum/const/oneOf — sin límites de pattern/format/numéricos) se resuelve al objeto validado. Se resuelve `null` cuando el hijo falla (filtra con `.filter(Boolean)`). Otros opts: `label` (visualización), `phase` (grupo de progreso) y anulaciones independientes de objetivo LLM `provider`/`model` (cualquiera puede proporcionarse solo). Cualquier otra cosa (`effort`/`isolation`/`agentType`) se rechaza de forma ruidosa.
- `pipeline(items, ...stages): Promise<any[]>` — ejecuta cada elemento a través de las etapas de forma independiente SIN barrera entre etapas (prefiérelo para trabajo de varias etapas). Cada etapa recibe `(prev, item, index)`. Un throw ordinario de una etapa reduce ese ELEMENTO a `null` y se salta sus etapas restantes.
- `parallel(thunks): Promise<any[]>` — ejecuta funciones sin argumentos de forma concurrente y espera TODAS (una barrera; úsalo solo cuando una etapa necesita de verdad todos los resultados anteriores juntos). Un thunk que lanza se resuelve a `null`.
- `phase(title)` — inicia una fase de progreso; `log(message)` — narra el progreso; `args` — la entrada `args` de la llamada de herramienta, verbatim.

Los hooks mal usados (argumentos incorrectos, opciones desconocidas, schemas no admitidos, topes superados) lanzan errores que SIEMPRE matan el script — nunca se disuelven en un `null` por elemento.

Restricciones: se aplican topes de concurrencia y de agent total; no se proporcionan APIs de sistema de archivos, red, temporizadores ni Node.js — los agents hacen el trabajo, el script solo los coordina. La ejecución corre en primer plano: esta llamada devuelve cuando todo el script termina.

```json
{
  "type": "object",
  "properties": {
    "script": {
      "type": "string",
      "description": "The plain-JS workflow script body (top-level await allowed; NO `export const meta` statement; end with `return <json-value>`)."
    },
    "meta": {
      "type": "object",
      "description": "The workflow identity block (plain JSON — never code).",
      "additionalProperties": true,
      "properties": {
        "name": {
          "type": "string",
          "description": "Short kebab-case workflow name."
        },
        "description": {
          "type": "string",
          "description": "One-line description of what the workflow does."
        },
        "whenToUse": {
          "type": "string",
          "description": "Optional guidance on when this workflow applies."
        },
        "phases": {
          "type": "array",
          "description": "Optional phase declarations matched by phase() calls.",
          "items": {
            "type": "object",
            "additionalProperties": true,
            "properties": {
              "title": {
                "type": "string",
                "description": "The phase title phase() calls match by exact string."
              },
              "detail": {
                "type": "string",
                "description": "Optional one-line description of the phase."
              },
              "provider": {
                "type": "string",
                "description": "Optional provider override this phase is expected to use."
              },
              "model": {
                "type": "string",
                "description": "Optional model override this phase is expected to use."
              }
            },
            "required": [
              "title"
            ]
          }
        }
      },
      "required": [
        "name",
        "description"
      ]
    },
    "args": {
      "type": "object",
      "description": "Optional JSON input exposed to the script as the `args` global (wrap a bare list as a field, e.g. {\"files\": [...]}).",
      "additionalProperties": true
    }
  },
  "required": [
    "script",
    "meta"
  ]
}
```


Fuente: [`packages/workflow/tool-workflow/src/index.ts`](../packages/workflow/tool-workflow/src/index.ts)

<a id="deepseek-aidsh-tool-web"></a>

## `@deepseek-ai/dsh-tool-web`

### `web_fetch`

Obtiene el contenido de una URL HTTP(S) concreta y lo devuelve decodificado a texto.

```json
{
  "type": "object",
  "properties": {
    "url": {
      "type": "string",
      "description": "The HTTP(S) URL to fetch."
    }
  },
  "required": [
    "url"
  ]
}
```


Fuente: [`packages/web/tool-web/src/index.ts`](../packages/web/tool-web/src/index.ts)

### `web_search`

Busca en la web información actual. Proporciona 1–4 consultas en el array obligatorio queries. Devuelve una respuesta resumida opcional y una lista de URLs de origen.

```json
{
  "type": "object",
  "properties": {
    "queries": {
      "type": "array",
      "description": "Required search queries; accepts 1–4 items and merges their results.",
      "items": {
        "type": "string"
      }
    }
  },
  "required": [
    "queries"
  ]
}
```


Fuente: [`packages/web/tool-web/src/index.ts`](../packages/web/tool-web/src/index.ts)

web_search y web_fetch mantienen la selección de provider detrás de ctx.web para que los schemas visibles para el modelo permanezcan estables entre cambios de backend.
