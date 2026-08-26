# @deepseek-ai/dsh-workflow-worker-thread

[English](README.md) | [中文](README.zh.md) | Español

Este paquete implementa `WorkflowEngine` con un hilo de trabajo de Node por ejecución. El worker ejecuta el script de orquestación; los agents hijos permanecen en el host y se alcanzan a través de `ctx.subagents` mediante un protocolo tipado host/worker.

La raíz del paquete exporta el plugin del motor predeterminado y su `Config`; el protocolo del worker, el runtime y los módulos de sesión permanecen privados a la implementación. La entrada operativa `./worker` sigue siendo el objetivo de spawn del motor.

La división tiene un propósito primario: un bucle de script síncrono no puede bloquear el event loop del harness, y un script que ignora la cancelación puede terminarse junto con su worker. No es un sandbox de seguridad.

## Frontera de confianza y aislamiento

Los scripts de flujo de trabajo los escribe el modelo y tienen la misma premisa de confianza que el acceso bash existente del modelo. `node:vm` dentro de un worker es un mecanismo de conformación de API, no una frontera de seguridad: un script escapado puede recuperar capacidades de Node con los privilegios del proceso del host.

El worker aun así aporta contención útil:

- El trabajo de CPU del script y los giros síncronos se mantienen fuera del event loop del host.
- `worker.terminate()` le da a la disposición una parada final real.
- El worker arranca con un entorno vacío, salvo la plomería del loader sin construir, así que las credenciales del entorno no cruzan a través de `process.env`.
- Los mensajes host/worker usan datos de clonación estructurada, con validación de JSON simple en la frontera del script.

Un sandbox genuino para scripts no confiables requeriría un motor diferente tras el mismo seam de flujo de trabajo.

## Contrato del script

El `meta` del flujo de trabajo son datos aportados por el host, no texto de script evaluado. El motor valida su `name` y `description` obligatorios, rechaza campos desconocidos y comprueba el análisis del cuerpo antes de devolver una ejecución.

Dentro del worker, el script recibe `args` y estos hooks:

- `agent(prompt, { label, phase, schema, model })` inicia un subagente en el lado del host. Con un schema devuelve el valor estructurado; de lo contrario devuelve texto final. Un hijo ordinario fallido devuelve `null`.
- `parallel(thunks)` ejecuta thunks bajo el límite de concurrencia configurado.
- `pipeline(items, ...stages)` pasa `(previous, item, index)` sin barrera entre etapas.
- `phase(title)` y `log(message)` emiten narración de observador.

Las opciones desconocidas, los argumentos malformados, los schemas no admitidos, los topes disparados, los fallos de arranque de provider y los fallos de resultado de infraestructura son errores fatales del flujo de trabajo. No se inyectan intencionadamente temporizadores, API de sistema de archivos ni globales de Node, aunque el aviso de confianza anterior sigue aplicando.

## Secuencia de ejecución

`start()` valida el meta, analiza el cuerpo, resuelve una ruta de provider normalizada registrada y resuelve cualquier tope de hijos totales por ejecución antes de crear un worker o publicar `workflow/start`. Un `maxTotalAgents` solicitado debe ser un entero seguro positivo no mayor que el techo de despliegue configurado del motor. El modo fuente instala transformaciones de TypeScript mediante un bootstrap de data-URL; el modo construido pasa el hermano `lib/worker.cjs` como ruta de sistema de archivos porque el hook VFS de pkg espera CommonJS. Ambos funcionan bajo Node ordinario. Un handshake ready/go evita que una cancelación de señal de arranque que compite con el arranque del worker ejecute la porción síncrona inicial del script.

Para cada llamada `agent()`:

1. El worker envía `child-start` con un prompt y opciones de datos simples.
2. El host llama a la invalidación de provider de la petición de arranque, o en su defecto al provider configurado, a través de `SubagentRuntime.start` asíncrono, pasando el padre del flujo de trabajo y una señal canónica de aborto por ejecución. La elección de provider aplica a todos los hijos de esa ejecución y no es visible para el script.
3. Si el arranque rechaza, el host envía `child-start-error`; el arranque del provider ya alcanzó la quietud y no se emite ningún evento de ciclo de vida de hijo.
4. Si el arranque se cumple mientras el flujo de trabajo aún admite trabajo, el host registra la ejecución, observa `result` y luego envía `child-started`. Incluso un resultado ya asentado se reenvía después, preservando el orden arranque-antes-que-resultado.
5. El worker emite la narración emparejada `workflow/agent-start` y `workflow/agent-end` y solicita la disposición del hijo tras la recolección.

Los arranques de provider se rastrean por separado de los hijos publicados. Si la cancelación, la muerte del worker o el asentamiento normal del flujo de trabajo cierran la admisión mientras un arranque está pendiente, la señal compartida lo aborta. Un provider que aun así se cumpla tras el cierre es dispuesto por el host y nunca se anuncia al worker.

## Frontera de valores

Los valores que salen del script pasan por `materializeFromRealm`, que acepta datos JSON simples y sin pérdidas y rechaza prototipos exóticos, funciones, símbolos, ciclos, arrays dispersos, números no finitos y `undefined` anidado. El recorrido se ejecuta en el worker y define las claves de objeto como propiedades de datos para que `__proto__` no pueda mutar un prototipo.

Los resultados de los hijos se proyectan y se les toma una instantánea antes de cruzar del host al worker. Esta es una frontera de serialización real, de tipo proceso; es deliberadamente distinta de las cargas de eventos de flujo de trabajo y subagente de confianza del mismo proceso, que son valores inmutables prestados.

## Cancelación y disposición

`WorkflowRun.cancel()` registra la primera razón, indica al worker que cancele, aborta la única señal compartida por todos los hijos pendientes y publicados, y arma el temporizador `disposeGraceMs`. Los hooks del worker lanzan entonces `CANCELLED` en su próximo await. Si la ejecución sigue sin asentarse en la fecha límite, el host la resuelve como cancelada, empareja los eventos de ciclo de vida de hijos varados y termina el worker.

El seam de subagente tiene un único canal de cancelación: la señal de la petición. No hay un RPC separado de cancelación de hijo. El desmontaje de hijos publicados usa `run.dispose()`; los arranques pendientes de provider siguen siendo propiedad del provider hasta que su promesa rechaza o se cumple.

El asentamiento normal también aborta los arranques pendientes y empieza a disponer cualquier hijo fire-and-forget publicado antes de que el resultado se asiente externamente. La condición de quietud del host incluye tanto los arranques pendientes como las disposiciones de hijos publicados, así que la limpieza no olvida una transacción de arranque asíncrono.

`dispose()` es idempotente. Cancela la ejecución, inicia de inmediato la disposición dirigida por el host, espera el resultado más la quietud de los hijos hasta la misma gracia, termina el worker incondicionalmente y realiza un barrido final de supervivientes. La disposición por hijo está memoizada, de modo que el RPC del worker, la cancelación del host, la limpieza por muerte y la disposición pública confluyen en una sola operación.

## Desenlace y garantías de eventos

El desenlace terminal es primero-en-ganar en los puntos de reclamación del host. Una cancelación externa aceptada prevalece sobre un resultado posterior del worker no cancelado; un resultado o una muerte del worker que reclama primero no puede reescribirse mediante callbacks de limpieza reentrantes.

El error del worker, el fallo de mensajes o la salida prematura cierran la admisión de mensajes antes de la limpieza, y luego resuelven `error` salvo que la cancelación ya sea dueña de la ejecución. Los mensajes tardíos en cola no pueden crear hijos ni narrar tras esa frontera lógica.

El host mantiene un libro de arranques de hijos reenviados. Un worker correcto aporta sus finales; la muerte o la terminación forzosa sintetiza cualquier final ausente como cancelado. Cada `workflow/agent-start` reenviado queda así emparejado exactamente una vez, aunque la limpieza tras un resultado del flujo de trabajo ya llegado pueda completarse después.

## Configuración

| Clave | Predeterminado | Significado |
|---|---|---|
| `provider` | `spawn` | Provider de subagentes del lado del host usado por `agent()`. |
| `maxConcurrentAgents` | `0` | Tope de `agent()` concurrentes; `0` se resuelve desde el paralelismo de CPU disponible. |
| `maxTotalAgents` | `1000` | Total de llamadas `agent()` en una ejecución. |
| `maxItemsPerCall` | `4096` | Elementos aceptados por una llamada `parallel()` o `pipeline()`. |
| `syncTimeoutMs` | `5000` | Tiempo de espera agotado de la VM para la porción síncrona inicial del script. |
| `disposeGraceMs` | `5000` | Cota antes del asentamiento forzoso/terminación y para la disposición pública. |

Un consumer propietario puede fijar `WorkflowStartRequest.subagentProvider` y `WorkflowStartRequest.maxTotalAgents` para una ejecución. Son política a nivel de motor, no hooks de script ni opciones orientadas al modelo; la herramienta `workflow` ordinaria deja ambas sin fijar. Un tope de hijos totales por ejecución puede bajar pero nunca subir el techo `maxTotalAgents` configurado.

## Experiencia del modelo

### Peticiones de los agents hijos

#### Lo que ve el modelo

Cada llamada `agent()` del script envía su prompt literal y el modelo opcional o el schema de salida estructurada a un provider de subagentes. Cada hijo ve el contexto propio de ese provider; la narración de fases y logs queda en los eventos de observador.

#### Efecto de tokens

Se pagan potencialmente muchos contextos de hijos independientes, acotados por `maxConcurrentAgents`, `maxTotalAgents` y `maxItemsPerCall`; nunca se unen directamente al historial del padre.

#### Efecto de KV Cache

Independiente de la caché de peticiones del padre y de los hijos hermanos. Cada hijo solo puede reutilizar un prefijo byte-idéntico bajo su propio provider, modelo, prompt y schema; su historial posterior crece solo con appends.

### Resultado de la herramienta padre, indirectamente

#### Lo que ve el modelo

A través de [`dsh-tool-workflow`](../tool-workflow/README.es.md), el éxito expone solo el valor JSON final materializado y el recuento de hijos en el envoltorio de ese consumer. Este motor aporta errores estables que incluyen `workflow script does not parse: <error>`, `invalid meta: <violations>`, `agent() requires a non-empty prompt string`, `agent() could not start a child: <error>`, `child agent run failed: <error>` y sus mensajes exactos de validación de `parallel()`, `pipeline()`, `phase()`, opciones, schemas y frontera JSON. Las salidas intermedias de los hijos están disponibles para el script pero no para el modelo padre.

#### Efecto de tokens

Cero tokens directos del padre por este motor. El tamaño del resultado final está acotado por el consumer de la herramienta y se retiene hasta la compactación.

#### Efecto de KV Cache

Solo append; el contenido recién visible sigue al prefijo reutilizable de la petición y no invalida las entradas existentes de la caché KV.

## Limitaciones conocidas y trabajo diferido

- **El worker/vm no es una frontera de seguridad** — el código escrito por el modelo puede escapar de `node:vm` y alcanzar la autoridad de proceso del worker; un despliegue con código hostil necesita un motor de proceso separado o de contenedor.
- **Se paga un hilo de trabajo por ejecución** — no hay pool, runtime caliente ni caché de scripts entre ejecuciones.
- **No se inyectan temporizadores, sistema de archivos ni red del entorno, pero el código escapado aun así puede alcanzar Node** — los globales ausentes son API de portabilidad, no contención.
- **La terminación solo puede reportar arranques observados por el host** — `agentsStarted` excluye las llamadas del lado del worker aún en cola tras la concurrencia cuando una terminación forzosa las vuelve incognoscibles.
- **Los errores entre reinos fallan `instanceof Error` dentro de los scripts** — los autores de flujos de trabajo deben ramificar sobre campos estables como `name` y `code`.
