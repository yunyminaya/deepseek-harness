# @deepseek-ai/dsh-tool-subagent-control

[English](README.md) | Español

Las herramientas opcionales con nombre global `send_message`, `interrupt_agent` y `list_agents` son adaptadores delgados sobre `ctx.subagents`. Las instancias de `@deepseek-ai/dsh-tool-subagent` vinculadas a un provider registran herramientas de delegación distintas por transporte; este paquete cargado por separado registra las herramientas de control compartidas una sola vez, así que varias herramientas de delegación nunca registran controles globales duplicados. El plugin raíz registra `send_message` e `interrupt_agent` y solo requiere `subagents`; el plugin `./list-agents`, cargable por separado, registra `list_agents` y declara `subagents` más `agents` como dependencias de tiempo de carga. Sus lecturas de catálogo requieren además el almacén de sesiones y el registro de proyecciones en el momento de la llamada, pero ningún servicio de consulta. Un despliegue puede conservar las herramientas raíz omitiendo la herramienta de listado. Ninguna presencia de herramienta determina si una herramienta de delegación inicia trabajo continuable. Estas herramientas son dueñas solo de la dirección padre-a-hijo; el instalado de forma independiente [`@deepseek-ai/dsh-tool-subagent-report`](../tool-subagent-report/README.es.md) es dueño de la dirección hijo-a-padre.

La herramienta no hace ningún enrutamiento de ciclo de vida — la residencia y la reanudación en frío pertenecen al servicio de subagente. Pasa `exec.agent` como el padre en vivo exacto que autoriza la entrega y registra cada fuente de mensaje como `{ kind: 'coordinator', senderSessionId: parent.id }`, que el servicio retiene pero nunca trata como autoridad. Cada mensaje se convierte en el siguiente turno FIFO del subagente mediante `Agent.followup()`: si el hijo sigue trabajando, el mensaje espera a que su turno actual termine, así que no puede redirigir trabajo ya en marcha. La herramienta reenvía su señal de ejecución, que es dueña de la admisión solo hasta la aceptación en el buzón; una vez que el hijo acepta el mensaje, el turno aceptado no se puede cancelar a través de esta herramienta. Esta llamada no devuelve ninguna respuesta del hijo — su transcript por ese id es la fuente de lo que hizo — y un hijo con `report` envía contenido por iniciativa propia como mensaje separado al padre. Un fallo de entrega se convierte en un resultado de herramienta con error que indica que el mensaje no se entregó.

`interrupt_agent(agent_id)` pasa `exec.agent` como la autoridad de ancestro en vivo exacta para `ctx.subagents.interrupt()`: el objetivo puede ser un hijo directo o un descendiente más profundo, y el servicio — nunca esta herramienta — verifica al llamador contra el linaje registrado de la Activación objetivo. Solo se detiene el turno actual del objetivo (`keepInbox`): los mensajes en cola permanecen en espera hasta un `send_message` posterior, los descendientes publicados siguen ejecutándose y el hijo sigue disponible para continuaciones. La llamada devuelve en cuanto la solicitud de parada se acepta, sin esperar la quiescencia del objetivo; un objetivo ausente o ya asentado es un no-op aceptado, mientras que los llamadores que son sí mismos, hermanos, obsoletos o no ancestros se convierten en resultados con error.

`list_agents` toma un argumento `scope` opcional, deriva el id de raíz del agent llamante y proyecta el catálogo del servicio a los hijos continuables sin cursor. El ámbito `children` por defecto lee `ctx.subagents.listChildren()`; `descendants` lee `ctx.subagents.listDescendants()`, cuyo recorrido de un único corpus cruza sesiones ordinarias e hijos one-shot y representa las filas supervivientes en preorden estable con `parent=<id> depth=<n>`. La anotación `parent` es el id de sesión persistente del padre directo y puede nombrar una sesión ordinaria omitida de la salida. Para el agent llamante, solo las entradas de hijo de profundidad 1 son candidatas a `send_message`; las entradas de hijo más profundas son candidatas solo a `interrupt_agent`. El estado viene del registro de Agents en vivo: `running` (driver activo), `idle` (residente entre turnos, posiblemente esperando a agents que inició) o `ready` (solo almacenamiento y reanudable en lugar de terminal). El resultado del servicio también contiene subagentes one-shot respaldados por sesión para Consumers como una UI, pero esas entradas se omiten de esta herramienta de modelo porque no pueden aceptar `send_message`. Los diagnósticos permanecen visibles, con posiciones en el ámbito de descendientes. La identidad persistente y el modo vienen del descriptor de cada hijo, mientras que la autoridad en tiempo de entrega y las comprobaciones de propiedad de la Activación siguen siendo del servicio.

## Experiencia del modelo

### Schema de la herramienta

#### Lo que ve el modelo

Los [schemas](../../../docs/tool-catalog.es.md#deepseek-aidsh-tool-subagent-control) generados: `send_message` toma `subagent_id` y `message`, describiendo que el mensaje se convierte en el siguiente turno del subagente, que esta llamada no devuelve ninguna respuesta del subagente y que un fallo significa que el mensaje no se entregó; `interrupt_agent` toma `agent_id`, describiendo que solo se detiene el turno actual, que los mensajes en cola quedan en espera, que los descendientes siguen ejecutándose y que la aceptación precede a la parada real; `list_agents` toma el enum `scope` opcional.

#### Efecto en tokens

Coste de schema fijo por solicitud del padre.

#### Efecto en KV Cache

Estable en prefijo; el schema no cambia en runtime.

### Resultado de interrupción

#### Lo que ve el modelo

`interrupt requested for agent <agent_id>` al aceptarse. Un llamador no autorizado — sí mismo, hermano, obsoleto o no ancestro — es un resultado con error que nombra el rechazo; un objetivo ausente o asentado sigue representando la línea de aceptación.

#### Efecto en tokens

Una confirmación breve por llamada; la cancelación del turno interrumpido solo es visible en el transcript propio del hijo.

#### Efecto en KV Cache

De solo añadido; cada resultado sigue el prefijo de solicitud reutilizable.

### Resultado de entrega

#### Lo que ve el modelo

`message queued as the next turn for subagent <subagent_id>` al aceptarse; la salida canónica lleva el `messageId` aceptado. Un fallo — un hijo no autorizado o desconocido, un hijo sin descriptor que no se puede reanudar o una admisión rechazada — es un resultado con error cuyo mensaje indica que el mensaje no se entregó.

#### Efecto en tokens

Una confirmación breve por llamada; la respuesta del hijo nunca vuelve a través de esta llamada. Un `report` concedido por separado puede añadir contenido seleccionado al historial del padre.

#### Efecto en KV Cache

De solo añadido; el contenido recién visible sigue el prefijo de solicitud reutilizable y no invalida las entradas existentes de la caché KV.

### Resultado de listado

#### Lo que ve el modelo

Una línea por hijo continuable en orden de catálogo estable: `<id> [<status>] — <label>` (`running` = driver activo, `idle` = residente entre turnos, `ready` = solo almacenamiento; reanudable en lugar de terminal, no un resultado esperando a ser recogido — un hijo directo en ese estado puede reanudarse con `send_message`), más `<id> [diagnostic: <reason>]` para un candidato que no se pudo leer (`corrupt`, `unsupported` o `unavailable`). El ámbito `descendants` inserta ` parent=<id> depth=<n>` antes del guion de la etiqueta en cada línea, en preorden. Los hijos one-shot están ausentes a propósito; `(no subagents)` significa que ningún hijo continuable ni diagnóstico sobrevivió a la proyección. Los diagnósticos nunca exponen el contenido de los descriptores.

#### Efecto en tokens

Crece linealmente con los hijos continuables listados — todo el árbol bajo el ámbito `descendants`; no hay cursor ni tope, así que los padres de larga vida con muchos hijos persistidos pagan la lista completa en cada llamada.

#### Efecto en KV Cache

De solo añadido; cada resultado sigue el prefijo de solicitud reutilizable.

## Limitaciones conocidas y trabajo diferido

- **Un mensaje en cola no tiene resultado independiente** — la aceptación devuelve solo su `messageId` de buzón; el trabajo del hijo aterriza en la Session persistente del hijo y nunca se recoge a través de esta herramienta. Un hijo con `report` concedido puede enviar contenido seleccionado por separado, pero ese mensaje no es el resultado de esta llamada.
- **Sin direccionamiento del turno actual** — cada mensaje abre un turno FIFO posterior, así que un mensaje enviado mientras el hijo trabaja se ejecuta solo después de que su turno actual termina y no puede redirigirlo.
- **El listado es una instantánea, no una promesa de entrega** — puede competir con la publicación, la liberación o un mensaje posterior, y otro proceso puede activar un hijo que este proceso informa como `ready`; la exactitud entre procesos requiere un lease compartido. `interrupt_agent` realiza él mismo la comprobación autoritativa del linaje en vivo, así que un descubrimiento obsoleto no puede conceder autoridad.
- **Sin paginación ni borrado** — se devuelve el conjunto completo ordenado de forma estable, y los hijos persistidos permanecen listados mientras sus sesiones sigan en la persistencia; un límite a nivel de servicio o una operación de borrado es una decisión de producto posterior.
