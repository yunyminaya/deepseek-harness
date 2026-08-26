# @deepseek-ai/dsh-workflow

[English](README.md) | [中文](README.zh.md) | Español

El seam de flujo de trabajo (`ctx.workflowEngine`) ejecuta un script de orquestación escrito por el modelo que puede hacer fan-out de subagentes. El seam define los contratos de script, ejecución, resultado, error y eventos; un motor decide cómo aislar y ejecutar el script.

`@deepseek-ai/dsh-workflow-worker-thread` es el motor actual y `@deepseek-ai/dsh-tool-workflow` es el consumer orientado al modelo. Un futuro motor de proceso o sandbox puede sustituir la implementación sin cambiar la herramienta.

La raíz del paquete es la cara Host. El subpath `@deepseek-ai/dsh-workflow/types` apto para navegador contiene identidades de ejecución, metadatos, resultados y cargas de ciclo de vida de solo-observación sin importar `Agent`, servicios de Cordis ni declaraciones de contexto Host; las `WorkflowStartRequest` y `WorkflowRun` exclusivas de Host viven tras la raíz del paquete.

## Contrato de servicio y de ejecución

`WorkflowEngine.start(request): WorkflowRun` valida lo suficiente de forma síncrona como para rechazar un bloque meta malformado, un script no analizable, una ruta de provider no disponible o un límite por ejecución no admitido antes de que exista una ejecución. Una vez devuelta, `WorkflowRun.result` nunca rechaza: los fallos de ejecución se resuelven con `stopReason: 'error'` y la cancelación se resuelve con `cancelled` dentro de la gracia acotada del motor.

Una ejecución es propiedad del tenedor. La descarga del plugin del motor impide nuevos arranques pero no revoca las ejecuciones aceptadas. El tenedor debe llamar a `dispose()` en todos los caminos; la disposición cancela el trabajo restante y alcanza o abandona la quietud dentro de la cota documentada.

`WorkflowStartRequest` contiene `{ meta, script, args?, subagentProvider?, maxTotalAgents?, parent, signal? }`. `parent` atribuye cada agent hijo al agente que invoca. `subagentProvider` opcionalmente enruta a todos los hijos de esa ejecución sin exponer la elección de provider al script; su omisión usa el provider configurado del motor. `maxTotalAgents` opcionalmente baja el techo de despliegue del motor para una ejecución y es igualmente invisible para el script. Una implementación rechaza rutas y límites inválidos síncronamente. `meta` y `args` son datos simples, no fragmentos de script.

`WorkflowRun` expone `{ id, meta, result, cancel(reason?), dispose() }`. `WorkflowResult` contiene `{ value, stopReason, error?, agentsStarted }`; `value` es datos JSON simples o `null`.

## Eventos

Los eventos del flujo de trabajo son de solo observación. Llevan `WorkflowRunInfo` (`id` más `meta`) en lugar de la ejecución viva, así que los listeners no pueden adquirir autoridad de cancelación o disposición.

- `workflow/start` / `workflow/end` emparejan la ejecución.
- `workflow/phase` y `workflow/log` exponen la narración del script.
- `workflow/agent-start` / `workflow/agent-end` emparejan cada llamada hija por `seq`; un hijo cuyo arranque asíncrono de provider rechaza no emite ninguno.

Las cargas de eventos del mismo proceso son valores inmutables prestados. Cada listener está contenido de forma independiente: un lanzamiento síncrono o una promesa devuelta rechazada se registran sin ahogar a sus pares ni cambiar la ejecución.

## Disciplina de fallos

`WorkflowError` lleva un código y una bandera `fatal`. Los errores fatales siempre escapan de `parallel()` y `pipeline()` en lugar de convertirse en un `null` ordinario por elemento:

- `SCRIPT_PARSE` / `META_INVALID` — el flujo de trabajo no puede arrancar.
- `INVALID_ARGUMENT` / `UNSUPPORTED_OPTION` / `UNSUPPORTED_SCHEMA` — una llamada a un hook viola el contrato del motor.
- `AGENT_CAP` / `ITEM_CAP` — se superaron los límites de seguridad configurados.
- `AGENT_START` — el arranque asíncrono del provider rechazó.
- `AGENT_RESULT` — el resultado de un hijo publicado rechazó con un fallo de infraestructura.
- `RESULT_UNSERIALIZABLE` — un valor de script/worker no son datos JSON simples.
- `CANCELLED` — la cancelación es dueña de la ejecución y los hooks pendientes/futuros rechazan.

Un hijo que se resuelve con normalidad con una razón de parada no completada no es una excepción de infraestructura: `agent()` devuelve `null`, permitiendo que el script gestione un fallo ordinario de hijo.

## Experiencia del modelo

Indirectamente, a través de `dsh-tool-workflow` y un motor de flujo de trabajo, que crean peticiones de agents hijos y devuelven un resultado de herramienta padre retenido.

#### Efecto de KV Cache

Sin invalidación directa; el consumer nombrado es dueño de cualquier cambio de prefijo de petición.

## Limitaciones conocidas y trabajo diferido

- **Solo recolección en primer plano** — el llamador es dueño de una ejecución viva y la espera; el arranque/sondeo en segundo plano, los handles de desbordamiento y la recolección desacoplada quedan diferidos.
- **Sin journaling ni reanudación** — los scripts, el progreso de los hijos y los valores intermedios no se guardan en puntos de control, así que un reinicio de proceso no puede continuar una ejecución.
- **Sin flujos de trabajo guardados o anidados** — el seam arranca solo scripts aportados por el llamador, y un script de flujo de trabajo no recibe ningún hook `workflow()` para orquestación recursiva.
- **Sin vocabulario de presupuesto de tokens** — los motores acotan concurrencia, elementos e hijos, pero ni la petición ni el resultado contabilizan tokens de modelo entre hijos.
- **Las ejecuciones son propiedad del tenedor, no rastreadas por el servicio** — descargar el motor no descubre handles vivos independientes; cada consumer debe disponer la ejecución que arrancó.

Ver la [Agent Note de dynamic-workflows](../../../.agents/notes/implemented/feature/2026-07-05-dynamic-workflows.es.md) para la API diferida de flujos de trabajo.
