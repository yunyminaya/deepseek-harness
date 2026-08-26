# @deepseek-ai/dsh-tool-subagent-report

[English](README.md) | Español

La herramienta `report` opcional con ámbito de hijo es un adaptador delgado sobre `ctx.subagents.reportFrom()`. Da a cada hijo continuable en proceso un canal de retorno al Agent que lo inició e instala la sección de prompt que instruye al hijo a usarla. El paquete registra una contribución de configuración de hijo continuable en lugar de una herramienta global, así que la herramienta y su guía existen solo dentro de esos hijos. Las raíces, los subagentes one-shot, los providers de subagente remotos, los ámbitos hermanos y la ejecución de herramientas sin agent nunca la presentan ni la ejecutan. Instalar este paquete concede solo esa capacidad con ámbito de hijo; la dirección padre-a-hijo sigue siendo el independiente [`@deepseek-ai/dsh-tool-subagent-control`](../tool-subagent-control/README.es.md), y el modo continuable no depende de ninguno de los dos paquetes.

La sección de prompt `tool:report` con ámbito de hijo instruye al hijo a llamar a `report` una vez antes de terminar, con una respuesta autocontenida, y antes cuando un hallazgo parcial cambia lo que el padre debería hacer después. La instrucción es guía, no imposición: el mecanismo sigue aceptando cero o muchas llamadas en un turno, y ninguna ruta de runtime rechaza a un hijo que nunca informa. Una llamada con éxito ni concluye el turno, ni asienta la Activación, ni impide continuaciones posteriores del padre, y terminar un turno nunca informa automáticamente. La herramienta no acepta destinatario: `exec.agent` es el Agent en vivo exacto del remitente y la credencial de autoridad, y el servicio deriva el único destinatario del `parentSession` persistente de ese hijo. El éxito devuelve el `MessageId` estable del mensaje aceptado por el padre, no un acuse de lectura, un id de ocurrencia de buzón, una confirmación en el log del padre, un recibo de finalización de turno ni un vaciado de persistencia. Un padre ausente del registro hace fallar la llamada con `direct parent is not live; report was not delivered` — la presencia en el registro gobierna la resolución del padre, y un padre registrado que ya está en la liberación propiedad del host sigue aceptando mientras su log admite añadidos. El servicio no realiza ninguna inyección, reanudación en frío del padre ni escritura de buzón sin conexión; el transcript persistente del hijo sigue siendo la fuente de recuperación, y una llamada de herramienta fallida no prueba la no entrega (un veto posterior de `tools/post-execute` puede hacer fallar una llamada cuyo report ya fue aceptado).

`reportDelivery` selecciona la programación del padre para cada report aceptado. `next-step` (el valor por defecto) usa `parent.steer()`: un padre en ejecución recibe el report en su límite de paso seguro más cercano, mientras que un padre inactivo inicia un turno. Los reports aceptados en secuencia comparten el FIFO de next-step, incluido el aviso de asentamiento redactado después por el gestor, así que el padre no puede observar el asentamiento antes que un report anterior; los reports que esperan juntos entran en un único lote reclamado. `quiet` usa `parent.inject()`, añadiendo el mismo contexto de next-step sin despertar a un padre en espera. Esta es política de programación del despliegue, así que el schema orientado al modelo no puede seleccionarla ni anularla por llamada.

El registro con ámbito local sobrevive deliberadamente al `toolFilter` global del hijo, así que una lista blanca de delegación no puede eliminar el único canal de retorno. Un despliegue que requiera un hijo sin canal de retorno omite este paquete.

El cuerpo de la contribución se exporta como `installReportTool(childCtx, ctx, delivery)` para que los Consumers de inspección puedan instalar `report` y su guía en un ámbito de hijo recién creado, y devuelve el único disposer que revoca ambos. El catálogo de herramientas generado usa esa ruta porque el registro global no puede exponer un schema local al ámbito. La composición de producción sigue entrando por `apply()`; el registro de contribuciones del seam de subagente permanece privado.

## Experiencia del modelo

### Schema de la herramienta

#### Lo que ve el modelo

El [schema `report`](../../../docs/tool-catalog.es.md#deepseek-aidsh-tool-subagent-report) generado: un `output` string requerido. Su descripción indica que el hijo debe informar una vez antes de terminar, que el report llega solo al Agent que inició al hijo y que no termina el turno. No lleva ningún parámetro de destinatario ni de modo de entrega. La sección de prompt `tool:report` separada repite la obligación fuera del schema, donde un hijo que ignora las descripciones de herramientas aún la lee.

#### Efecto en tokens

Coste fijo de schema y de sección de prompt por solicitud de hijo continuable, y ninguno en las solicitudes de cualquier otro Agent.

#### Efecto en KV Cache

Estable en prefijo dentro de un hijo; ni el schema ni la sección cambian en runtime. Eliminar el paquete revoca ambos de los hijos residentes, lo que cambia su siguiente prefijo de solicitud.

### Resultado de report

#### Lo que ve el modelo

`report accepted by the agent that started you as message <messageId>` al aceptarse; la salida canónica lleva el `messageId` estable. Un fallo de un remitente no autorizado, de un padre no disponible o de un ciclo de vida en cierre es un resultado con error. La descripción dice que una llamada fallida puede haber llegado igualmente porque un fallo posterior de `tools/post-execute` puede reemplazar el resultado después de que `reportFrom()` aceptó el mensaje.

#### Efecto en tokens

Una confirmación breve por llamada en el hijo que informa. El contenido informado se cobra además al padre: la entrega next-step se une a la siguiente solicitud en un turno de padre abierto o inicia un turno para un padre inactivo, mientras que la entrega silenciosa espera a otra entrada para despertar al padre.

#### Efecto en KV Cache

De solo añadido en el hijo. En el padre, el report encuadrado sigue el historial existente y preserva el prefijo reutilizable.

### Report visible para el padre

#### Lo que ve el modelo

Un mensaje de padre con rol de usuario encuadrado como `Background subagent <child-id> reported:` seguido del `output` exacto del hijo, con una fuente persistente `{ kind: 'subagent-report', senderSessionId: <child-id> }` que nombra al hijo.

#### Efecto en tokens

El `output` completo del hijo más el marco de una línea, sin tope por este paquete.

#### Efecto en KV Cache

De solo añadido; el report sigue el prefijo de solicitud reutilizable del padre. La entrega next-step despierta al padre y puede extender su turno abierto, mientras que la entrega silenciosa no lo despierta.

## Limitaciones conocidas y trabajo diferido

- **Un padre cuya liberación propiedad del host ya comenzó aún puede aceptar** — `AgentHandle.dispose()` cancela, espera la quiescencia y solo entonces deshace el ámbito y deja el registro; no expone ninguna señal de «liberación iniciada». Un report aceptado en esa ventana se añade al transcript del padre, pero ese padre no actuará sobre él en este proceso. Un padre propiedad del gestor de continuación rechaza el desmontaje del bosque a través del límite de admisión del gestor.
- **La aceptación es más débil que la entrega persistente** — no hay buzón persistente, clave de idempotencia, recibo de entrega, protocolo de reintento ni reivindicación de exactamente-una-vez. Un fallo de proceso después de que un lado registró la aceptación deja el resultado ambiguo, y un reintento externo puede duplicar el report.
- **Un report silencioso escenificado no es reconstruible de inmediato** — la aceptación devuelve su `MessageId` estable, pero la Session del padre reconstruye el contenido encuadrado solo después de que el contexto pendiente alcanza su límite de log ordinario.
- **La concesión espera a la siguiente Activación; la revocación es inmediata** — instalar este paquete después de que un hijo se vuelve residente concede `report` y su guía solo en la siguiente Activación de ese hijo, mientras que eliminar el paquete revoca ambos de los hijos residentes de inmediato.
- **El report anidado alcanza exactamente un borde hacia arriba** — un nieto informa a su padre hijo directo, nunca al coordinador de nivel superior, que debe informar explícitamente una actualización derivada más tarde.
- **Sin límite de frecuencia** — el modo `next-step` por defecto puede amplificar el trabajo del modelo cuando los hijos anidados informan con frecuencia, aunque los reports que esperan juntos comparten un paso; un despliegue que acepte reports no leídos a cambio de esa amplificación selecciona `quiet`.
