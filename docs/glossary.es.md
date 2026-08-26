# Glosario

[English](glossary.md) | [中文](glossary.zh.md) | Español

El vocabulario de dominio de DeepSeek Harness usa un término canónico por concepto. Los términos enlazan con sus entradas mediante anclas Markdown estándar; el detalle de implementación permanece en los README de los paquetes y en las Agent Notes.

## capability-seam

- **seam** — una *capacidad reemplazable* con tres roles: una **Service Definition** (el `Service` de Cordis que posee su `ctx.<key>` y sus tipos de vocabulario — una clase abstracta como `ShellExecutor`, o un registro concreto como `WebRuntime`, nunca una `interface` de TypeScript), uno o más **Service Providers** y uno o más **Consumers** que inyectan el servicio. `packages/shell` es el ejemplo canónico: `dsh-shell` (Service Definition), `dsh-bash-local` / `dsh-bash-sandbox` (providers) y `dsh-tool-bash` (Consumer). Los roles normalmente ocupan paquetes separados cuando evolucionan de forma independiente, pero un paquete puede poseer varios roles cuando son una sola preocupación (`dsh-llm` posee su Service Definition y su Consumer). El seam es la capacidad completa, nunca un solo rol; reserva el término para ese significado y nombra cada componente por su rol, clase, servicio, contrato o punto de extensión.

## agent-scope

- **scope** — la unidad de registro por agent: una contribución (herramienta, sección de prompt, variable, restricción, listener) es *global* (visible para todos los agents) o *con scope* (pertenece a exactamente una [scope key](#scope-key)). Dos niveles, planos: los registros con scope no se heredan hacia los subagentes; el comportamiento de subárbol se expresa con datos de [lineage](#lineage), nunca con estructura de scope.
- **scope key** — la identidad opaca por la que se indexa un scope, comparada por identidad de objeto. Convención del harness: un agent vivo es la clave de su propio scope. <a id="scope-key"></a>
- **contexto de agent (`agent.ctx`)** — el contexto con scope del agent; los registros a través de él son visibles para el scope y tienen la vida útil del scope (un mismo hecho impulsa ambas), y sus listeners participan en los despachos filtrados por scope de ese agent. Los eventos de sujeto-registro pueden permanecer deliberadamente sin filtrar bajo sus propios contratos de evento.
- **scope carrier** — el `thisArg` que porta un despacho filtrado por scope (construido por `scopeTarget`); su filtro admite a los listeners sin etiqueta más los del propio sujeto. Un carrier *sin sujeto* (sin clave) admite solo a listeners sin etiqueta.
- **scoped dispatch** — la regla: un evento sobre la actividad de un agent se despacha con el carrier de ese agent. Los eventos sobre un registro en sí (se añadió una herramienta) son de *sujeto-registro* y permanecen sin filtrar.
- **shadowing** — resolución de nombres donde gana el más específico: una herramienta/sección/variable con scope reemplaza a su gemela global del mismo nombre solo para ese scope. Es el mecanismo de persona por agent y de variante de herramienta por agent.
- **restricción / registro local al scope** — una restricción (`tools.restrict`) filtra el conjunto GLOBAL de herramientas para un solo scope (compón por intersección); los registros locales al scope se fusionan después de ese filtro. Una herramienta global filtrada está ausente del prompt Y se niega a ejecutarse, indistinguible de una que no existe.
- **setup window** — la slot de creación donde un creador compone el mundo con scope de un agent (`CreateAgentOptions.setup`): después de que existen el scope y el objeto del agent, pero antes de publicar el agent o la sesión, de que se dispare `agent/session-start` o de que se ensamble el primer prompt. El setup registra; nunca dirige al agent.
- **lineage** — hechos padre/hijo transportados como datos (`parentSession`, `delegationDepth` durable, `subagentDepth` en runtime); nunca afecta a la visibilidad. <a id="lineage"></a>

## objetivo

- **objetivo** — un objetivo de finalización durable adjunto a una sesión existente, con una fase `active` / `paused` / `blocked` / `complete` sujeta a revisiones y un tope de rondas de objetivo; `blocked` conserva un código de política y una explicación. Un objetivo es un estado, no un programador ni una conversación aparte; el registro de la sesión sigue siendo su fuente de verdad.
- **ronda de objetivo** — un ciclo de continuación admitido para el objetivo actual. El driver de la misma sesión materializa una ronda de objetivo como un [turno](#turn) originado por el objetivo, que puede contener cero o más pasos; los turnos humanos no relacionados de la misma sesión no consumen el tope de rondas de objetivo. <a id="goal-round"></a>
- **activación de objetivo** — permiso local al proceso para que un consumidor de continuación admita otra ronda de objetivo. La activación está `armed` o `disarmed`; está deliberadamente ausente de la reproducción durable, por lo que reanudar y bifurcar exigen una mutación de reanudación autorizada por un humano más adelante, a través de `/goal` o de la herramienta del modelo, antes de que comience el trabajo automático.

## comando humano

- **comando humano** — una instrucción con prefijo de barra que un adaptador orientado a humanos interpreta y ejecuta a través de `ctx.commands`, sin convertirse en un mensaje de modelo. Se distingue de una herramienta orientada al modelo y de la ejecución de comandos de shell a través de `ctx.shell`.
- **plano de comandos** — descubrimiento, análisis, despacho, cancelación y renderizado de resultados, propiedad de los adaptadores de UI y de los plugins de comandos. La salida de un comando es estado de la UI salvo que el manejador mute por separado un dominio durable.
- **comando de objetivo** — el comando humano `/goal` aportado por `dsh-command-goal`; observa o muta directamente el objetivo actual, mientras que el dominio del objetivo posee cada registro durable y visible para el modelo.

## jerarquía de bucles

- **turno** — un vaciado de la entrada admitida en una sesión, que termina cuando el modelo y sus herramientas se detienen o interviene una política terminal. <a id="turn"></a>
- **paso** — una solicitud de modelo más las ejecuciones de herramientas causadas por su respuesta; un turno contiene cero o más pasos. <a id="step"></a>
- **ronda** — una iteración de política exterior que contiene un turno, como una [ronda de objetivo](#goal-round) o un intento de Ralph con un agent nuevo. Los contadores de ronda pertenecen a esa política y no cuentan cada turno de una sesión. <a id="round"></a>

## Ralph

- **bucle Ralph** — una ejecución en primer plano de flujo de trabajo de agent nuevo hacia un objetivo inmutable. Es una política de herramienta orientada al modelo compuesta a partir de primitivas de flujo de trabajo y subagente, no un objetivo de la misma sesión, un modo de agent loop, un programador ni una funcionalidad genérica de script de flujo de trabajo. <a id="ralph-loop"></a>
- **ronda Ralph** — una sesión hija nueva en un [bucle Ralph](#ralph-loop). La hija no recibe semilla de conversación del padre ni de la hija anterior; el workspace compartido y un [traspaso Ralph](#ralph-handoff) acotado transportan el estado entre rondas. <a id="ralph-round"></a>
- **traspaso Ralph** — el informe estructurado normalizado y acotado que pasa de una ronda Ralph que continúa a la siguiente, con estado, resumen, evidencia, próximos pasos y texto de bloqueo. Complementa al workspace compartido en lugar de sustituirlo como autoridad. <a id="ralph-handoff"></a>
