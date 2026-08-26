# @deepseek-ai/dsh-tool-subagent

[English](README.md) | Español

La herramienta de delegación orientada al modelo sobre un provider configurado de `ctx.subagents`. Cambiar el provider cambia el transporte sin cambiar el contrato de ejecución.

## Selección de provider y ciclo de vida

Cada instancia del plugin vincula un `provider` a un `toolName`; el modelo no recibe ningún selector de provider. Carga otra instancia con nombre distinto para exponer otro transporte. La herramienta se registra solo mientras su provider existe, evitando dependencias de orden de carga entre hermanos y de recarga del provider. Su descripción sigue a `provider.inheritsParentContext`: los hijos frescos exigen prompts autónomos, mientras que los hijos bifurcados ya ven los turnos completados del parent.

Una llamada en primer plano pasa la señal de ejecución a través del arranque y de la ejecución, espera `run.result` y siempre espera `run.dispose()` antes de devolver. Solo `completed` devuelve el `{ kind: 'foreground', runId, output: JsonValue[] }` canónico, renderizado como el mismo texto final. El aborto, la negativa, el límite de tokens y otros fallos se convierten en resultados de herramienta con error cuyo mensaje contiene el titular del motivo de parada, un `SubagentResult.diagnostic` opcional autoría del provider, y después cualquier texto parcial del asistente preservado. El diagnóstico permanece separado de `SubagentResult.output`, así que una respuesta truncada nunca se informa como éxito ni se confunde con detalle de infraestructura. Si la recogida del resultado y el disposal rechazan ambos, el resultado con error preserva ambos fallos.

`backgroundMode` selecciona tanto la ruta de segundo plano como el predeterminado omitido de `run_in_background`. `one-shot` espera en primer plano por defecto; un `true` explícito registra una Task plain propiedad del parent y devuelve el `{ kind: 'background', jobId }` canónico, renderizado como `started background subagent job <id>`, incluso cuando el provider soporta hijos continuables. Las herramientas genéricas de tareas son dueñas de su estado posterior, recogida, cancelación y avisos; una Task fallida conserva el motivo de parada y el mismo diagnóstico opcional del provider en su detalle. `continuable` corre en segundo plano cuando el argumento se omite o es `true`; un `false` explícito espera el resultado en primer plano. Su ruta de segundo plano exige un provider con la capacidad `prepareContinuable`, llama a `ctx.subagents.startContinuable()` y devuelve `{ kind: 'continuable', subagentId }`, renderizado como `started subagent <childId>`. La ruta se resuelve en la aceptación del inbox: el hijo es dueño de sus propios turnos a partir de ahí, así que esta llamada ni espera ni recoge un resultado. El transcript del hijo bajo ese id sigue siendo la fuente de su salida detallada, y la herramienta global opcional `send_message` le envía más trabajo. El servicio de continuación entrega un aviso de asentamiento cada vez que la Activación del hijo termina, conteniendo su desenlace y cualquier mensaje final del asistente con independencia de `report`. Iniciar trabajo continuable no exige que `send_message` esté cargada. Consulta la [Agent Note de subagentes en segundo plano](../../../.agents/notes/implemented/feature/2026-07-08-background-subagent-tasks.es.md), la [Agent Note de subagentes continuables](../../../.agents/notes/implemented/feature/2026-07-28-continuable-subagent-conversations.md) y la [Agent Note de delegación en segundo plano primero](../../../.agents/notes/implemented/feature/2026-08-11-background-first-continuable-delegation.md).

`toolFilter` cambia la capa de herramientas globales del hijo pero no es un techo de autoridad derivado del parent. Consulta el [objetivo no de seguridad del ámbito del agent](../../../.agents/notes/implemented/architecture/2026-07-08-agent-scope-contexts.es.md#security-and-authority-are-non-goals).

## Configuración

| Clave | Significado |
|---|---|
| `provider` (obligatorio) | Nombre del provider (`spawn`, `fork`, `acp`, ...). |
| `toolName` | Nombre orientado al modelo, por defecto `subagent`; distinto para cada instancia cargada. |
| `enableRunInBackground` | Expone el modo de segundo plano, por defecto `true`; deshabilitarlo también rechaza las llamadas forzadas a segundo plano. |
| `backgroundMode` | Política de ciclo de vida del segundo plano, por defecto `one-shot`. `one-shot` fija las llamadas por defecto en primer plano; `continuable` las fija por defecto en segundo plano, exige la capacidad `prepareContinuable` del provider y devuelve un id durable del hijo sin requerir la herramienta de seguimiento. |
| `agentOptions` | `provider`, `model` y `maxTokens` positivo hijos, específicos del provider; el provider en proceso trata los valores explícitos como sobrescrituras de las opciones heredadas del parent. |
| `persona` | Persona por hijo; exige la capacidad `persona` del provider. |
| `toolFilter` | Restricción de herramientas globales por hijo; exige la capacidad `toolFilter`. |
| `maxDepth` | Tope absoluto de profundidad de delegación, por defecto `3` (`0` prohíbe delegar); un tope numérico exige la capacidad `depthLimit` y hace fallar el montaje sin ella. `'provider-managed'` no envía tope para un provider fuera de proceso cuyo presupuesto pertenece al harness del hijo. La herramienta sigue visible en el tope; cada intento de arranque comprueba la profundidad actual del agent que llama y devuelve un resultado de herramienta con error cuando se rechaza. |

## Concurrencia

Las llamadas en primer plano y en segundo plano son seguras para la concurrencia: las delegaciones de hermanos en un mismo mensaje del asistente se solapan bajo el pool rodante del loop (`maxParallelToolCalls`), y los resultados siguen comprometiéndose en orden del modelo. Los hijos trabajan en sus propias sesiones y una ejecución nunca muta la sesión del parent; la única escritura propiedad del parent de la forma one-shot en segundo plano — registrar una Task — es una inserción síncrona y conmutativa que tolera el despacho concurrente, así que las llamadas en segundo plano solapadas adquieren sus ids de trabajo en el orden de carrera de despacho. Coordinar los efectos en el workspace entre hermanos corresponde al modelo, exactamente como ya lo hace para los hijos en segundo plano y continuables. Consulta la [Agent Note de delegaciones paralelas de subagentes](../../../.agents/notes/implemented/feature/2026-08-09-parallel-subagent-delegations.es.md) y la [Agent Note de llamadas de herramienta paralelas](../../../.agents/notes/implemented/feature/2026-07-10-parallel-tool-call-execution.md).

## Experiencia del modelo

### Schema de la herramienta

#### Lo que ve el modelo

El [schema `subagent` por defecto generado](../../../docs/tool-catalog.es.md#deepseek-aidsh-tool-subagent) bajo el nombre configurado de esta instancia mientras su provider existe. La herencia de contexto del provider cambia las descripciones de la herramienta y del prompt. El modo de segundo plano habilitado añade `run_in_background`: el modo continuable documenta su predeterminado `true`, su aviso de asentamiento en runtime y la sobrescritura explícita de primer plano, mientras que el modo one-shot documenta su predeterminado `false` y el id de trabajo que se recoge con `job_output` o se detiene con `job_kill`. Mientras la herramienta está visible en el ámbito de un ensamblaje, una sección de system prompt `tool:<toolName>` le dice al modelo que arranque juntas las delegaciones continuables independientes, que siga trabajando mientras corren y que elija el primer plano solo cuando su siguiente acción dependa del resultado; una restricción de herramienta elimina tanto su schema como esta guía.

#### Efecto en tokens

Coste fijo de schema por petición del parent; cada instancia de provider añade un schema, y cada instancia continuable añade una sección corta de system prompt.

#### Efecto en KV cache

Estable como prefijo mientras las instancias de provider, los nombres, las descripciones y los schemas no cambien. El ciclo de vida del registro del provider puede invalidar la reutilización del parent desde la primera definición de herramienta cambiada.

### Resultado en primer plano

#### Lo que ve el modelo

La llamada retiene la descripción y el prompt. El éxito contiene solo el texto final del hijo; otros desenlaces se convierten en `Error: <stop reason>`, seguido de un diagnóstico seguro del provider cuando existe y después cualquier texto parcial del asistente. Los pasos intermedios del hijo permanecen fuera del parent.

#### Efecto en tokens

El prompt y el resultado permanecen en el historial del parent hasta la compactación; el contexto de trabajo del hijo permanece en el hijo.

#### Efecto en KV cache

Solo append; el contenido recién visible sigue al prefijo reutilizable de la petición y no invalida las entradas existentes de la caché KV.

### Resultado en segundo plano

#### Lo que ve el modelo

El arranque devuelve exactamente `started subagent <childId>` en el modo continuable configurado, o `started background subagent job <id>` en el modo one-shot configurado. En el modo one-shot la superficie genérica de tareas proporciona el estado posterior, la salida final, las respuestas de cancelación y los avisos; el detalle del estado fallido incluye el diagnóstico del provider cuando el resultado llevó uno. En el modo continuable esta herramienta no devuelve ningún resultado propio; el asentamiento del hijo alcanza al parent como un [aviso propiedad del servicio](../subagent/README.es.md#settlement-notice), una herramienta `send_message` cargada de forma independiente entrega los seguimientos, y el transcript del hijo bajo su id es la fuente de su salida detallada.

#### Efecto en tokens

El reconocimiento se retiene; una salida final one-shot entra en el historial del parent solo cuando se recoge o se inyecta, mientras que la salida de un hijo continuable nunca vuelve a través de esta herramienta — su aviso de asentamiento llega con independencia de cualquier resultado de herramienta.

#### Efecto en KV cache

Solo append; el contenido recién visible sigue al prefijo reutilizable de la petición y no invalida las entradas existentes de la caché KV.

## Limitaciones conocidas y trabajo diferido

- **Las ejecuciones en segundo plano no exponen ningún resultado a través de esta herramienta** — la salida final de una tarea one-shot se recoge a través de la superficie genérica de tareas, y la salida de un hijo continuable permanece en su propia sesión, leída por su id de subagente. El aviso de asentamiento declara cómo terminó ese hijo y lleva cualquier mensaje final del asistente, pero no es el valor de retorno de esta llamada y no puede esperarse aquí.
- **Los nombres duplicados entre instancias one-shot en espera se detectan tarde** (`TODO(subagent-dup-toolname)`) — las instancias continuables reservan su nombre de sección de prompt durante la aplicación del plugin, pero impedir la reversión del registro del provider para instancias one-shot en espera exige un registro de nombres intencionales.
- **La política del hijo es fija por instancia** — otro modelo, persona, filtro de herramientas o tope de profundidad exige otra herramienta con nombre distinto.
