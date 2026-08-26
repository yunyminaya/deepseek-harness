# @deepseek-ai/dsh-tool-ralph

[English](README.md) | [中文](README.zh.md) | Español

La herramienta `ralph` orientada al modelo ejecuta un flujo de trabajo fijo de primer plano que entrega un objetivo inmutable a una secuencia de agents hijos frescos. Demuestra una política de orquestación especializada como un plugin ordinario sobre [`ctx.workflowEngine`](../workflow/README.es.md) y [`ctx.subagents`](../../subagent/subagent/README.es.md): no se añade ningún modo Ralph ni bucle de agents frescos a `agent-loop`, y el dominio de objetivos de la misma sesión ([goal domain](../../goal/goal/README.es.md)) permanece independiente. La [Agent Note de Ralph](../../../.agents/notes/implemented/feature/2026-07-19-fresh-agent-ralph-workflow-tool.es.md) es dueña de la política y del trabajo diferido.

## Contrato

`ralph({ objective, maxRounds? })` espera la ejecución completa. El `maxRounds` de la configuración de despliegue es a la vez el valor predeterminado y el tope de una invalidación por llamada. Cada ronda de Ralph inicia un hijo a través de `subagentProvider`; ese provider debe existir, admitir salida estructurada e informar `inheritsParentContext: false`. El provider configurado se transporta como `WorkflowStartRequest.subagentProvider`, de modo que el script fijo no puede inspeccionar ni cambiar el enrutamiento y la herramienta `workflow` ordinaria escrita por el modelo no gana selector de provider. El tope de rondas resuelto también se transporta como `WorkflowStartRequest.maxTotalAgents`, coordinando el bucle fijo con el tope de hijos totales del motor; el motor rechaza un tope de Ralph superior a su techo de despliegue antes de publicar una ejecución.

Cada hijo recibe únicamente el objetivo inmutable, su ronda y tope actuales de Ralph, una instrucción de espacio-de-trabajo-compartido-como-autoridad y el handoff estructurado anterior. El espacio de trabajo es la memoria a largo plazo; no se siembra la conversación del padre ni las sesiones de hijos previos. Los informes llevan `status: continue | complete | blocked`, un resumen no vacío, evidencia, próximos pasos y texto del bloqueo. Las semánticas específicas de cada estado y el tope serializado `maxHandoffChars` se validan dentro del flujo de trabajo fijo y de nuevo en la frontera del consumer. Los informes inválidos, ausentes o sobredimensionados hacen fallar el flujo de trabajo en lugar de truncarse o confundirse con agotamiento del tope.

El resultado terminal correcto de la herramienta es `complete`, `blocked` o `budget-limited`, con el último informe acotado y el número de rondas iniciadas. El sobre canónico es `{ runId, agentsStarted, result }`; las etiquetas de finalización y de bloqueo en su renderizador Native dicen explícitamente que un worker reportó el desenlace, no una certificación independiente. `maxResultChars` acota únicamente ese texto renderizado, incluida su marca de truncamiento, sin alterar el informe validado del valor canónico ni el handoff entre rondas.

Un fallo ordinario de un hijo produce un error que nombra la ronda fallida y retiene el último handoff correcto cuando existe. Ralph no reintenta esa ronda. Los fallos fatales de arranque de provider, transporte, worker o flujo de trabajo siguen siendo errores del flujo de trabajo y pueden asentarse antes de que el script fijo pueda devolver un handoff. La cancelación también es un error; la salida parcial nunca es éxito.

## Ciclo de vida y cancelación

El agent del llamador es el padre de cada hijo fresco, preservando cwd y linaje sin copiar su conversación. `exec.signal` entra en el motor del flujo de trabajo y también se tiende un puente hacia `run.cancel()` por independencia de la implementación. La herramienta espera `run.result` y llama a `run.dispose()` en `finally`, así que un paso padre cancelado espera la terminación acotada del motor y la quietud de los hijos antes de volver.

## Intención de renderizado

La llamada pendiente es una tarjeta `generic` titulada `ralph`; el objetivo inmutable es su `rawInput`. El resultado conserva la tarjeta genérica. Ambas funciones de presentación dependen solo de los argumentos de la herramienta y del sobre ya asentado de la herramienta.

## Configuración

| Clave | Predeterminado | Significado |
|---|---|---|
| `subagentProvider` | `spawn` | Provider fresco de salida estructurada usado en cada ronda. |
| `maxRounds` | `256` | Predeterminado y tope de despliegue de una ejecución de Ralph. |
| `maxHandoffChars` | `16384` | Máximo de caracteres serializados en un informe de ronda. |
| `maxResultChars` | `16384` | Máximo de caracteres en el resultado padre completo correcto. |

Todos los valores de configuración se normalizan y validan cuando el plugin se aplica, incluida la aplicación directa fuera de la normalización de schema del Loader. Las capacidades del provider se resuelven inmediatamente antes de cada llamada porque el registro de providers puede cambiar bajo el ciclo de vida del plugin y el HMR.

## Experiencia del modelo

### Prompt del sistema

#### Lo que ve el modelo

Toda petición padre dentro del ámbito de registro de este plugin recibe la guía fija de enrutamiento siguiente.

##### Guía de Ralph

```markdown
Use the ralph tool ONLY when the direct human explicitly asks for a Ralph loop or fresh-agent iterative execution. Each Ralph round starts a fresh child with no conversation seed and uses the shared workspace as durable memory. Completion and blockers are worker reports, not independent evaluation. Use same-session goal tools for ordinary long-running objectives, and plain subagents or workflowEngine for bounded delegation and fan-out.
```

#### Efecto de tokens

Pequeño coste fijo de guía por petición mientras el plugin está activo.

#### Efecto de KV Cache

Prefijo-estable mientras el ámbito del plugin y el texto de la guía no cambien. La activación o la disposición puede invalidar la reutilización de esta sección del prompt.

### Schema de la herramienta

#### Lo que ve el modelo

El schema [`ralph`](../../../docs/tool-catalog.es.md#deepseek-aidsh-tool-ralph) generado expone una cadena `objective` obligatoria y un número `maxRounds` opcional. La elección de provider, el tamaño del handoff, el schema de informe, el script del flujo de trabajo y el comportamiento de orquestación son propiedad del despliegue y están ausentes del schema de llamada.

#### Efecto de tokens

Pequeño coste fijo de schema en cada petición donde la herramienta es visible.

#### Efecto de KV Cache

Prefijo-estable mientras la definición y la visibilidad no cambien.

### Peticiones de los hijos y resultado padre

#### Lo que ve el modelo

Cada hijo ve el prompt fijo autónomo de ronda más el contrato de captura de salida estructurada. El padre ve solo la llamada original y un resultado terminal que contiene un estado reportado por un worker, el número de rondas y el informe final formateado; los mensajes e informes intermedios de los hijos no entran en la conversación del padre. Un hijo ordinario fallido produce en cambio un error con su número de ronda y, tras la primera ronda, el último handoff correcto.

#### Efecto de tokens

Cada ronda paga un contexto de hijo fresco. `maxHandoffChars` acota el estado entre rondas y `maxResultChars` acota de forma independiente el texto padre completo correcto; el trabajo de los hijos queda fuera del contexto padre.

#### Efecto de KV Cache

Cada hijo fresco tiene una caché de peticiones independiente. El resultado padre se añade después del prefijo reutilizable de la petición.

## Limitaciones conocidas y trabajo diferido

- **La finalización es autodeclaración del worker** — no hay evaluador ni verificador independiente que decida si el objetivo está realmente completo; la política de evaluación y la continuación dirigida por evaluador quedan diferidas.
- **Solo primer plano** — no hay id de trabajo, recolección en segundo plano, punto de control de reanudación de proceso, planificador ni política de arranque por reloj de pared.
- **El espacio de trabajo es la única memoria a largo plazo entre rondas** — un informe acotado es el handoff explícito, y el razonamiento conversacional no confirmado desaparece con cada hijo.
- **Una ronda es un hijo fresco** — no hay fan-out dentro de la ronda, cambio de modelo/provider, contexto de bifurcación ni provider seleccionado por llamada del modelo.
- **El fallo ordinario de un hijo es terminal para la ejecución** — el script fijo reporta la ronda fallida y el último handoff correcto pero no reintenta; los fallos fatales de la infraestructura del flujo de trabajo pueden terminar antes de que se devuelva ese estado.
- **Solo el recuento de rondas acota el esfuerzo agregado** — los presupuestos de tokens, precio y tiempo transcurrido quedan diferidos.
