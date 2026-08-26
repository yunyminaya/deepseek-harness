# Agent Note: La entrega de la liquidación pertenece al gestor de continuación

Status: implemented

[English](2026-08-06-manager-owned-subagent-settlement-delivery.md) | Español

## Problema

La delegación en segundo plano continuable era la única operación asíncrona que un modelo podía iniciar pero de la que no podía alcanzar el final. Cada otra forma tiene un primitivo de recuperación o un valor de retorno: un comando bash en segundo plano y un subagente (subagent) en segundo plano de un solo uso se liquidan ambos a través de un Task en el que `job_output(wait: true)` puede bloquearse, un workflow y un subagente en primer plano devuelven su resultado al llamador. Un hijo continuable en segundo plano devolvía solo su id durable, y no existía nada en lo que un padre pudiera esperar o que se le entregara.

[La obligación de report](2026-08-06-continuable-child-report-obligation.es.md) cerró la mitad cooperativa de esa carencia instruyendo al hijo a reportar antes de terminar. La instrucción no puede cerrar el resto. Un hijo detenido por un techo de tokens, un fallo del modelo, una cancelación o un teardown nunca llega al punto en que podría cumplir — no raramente, sino nunca — y esos son precisamente los finales que un padre en espera más necesita conocer. Los síntomas observables aguas abajo eran padres haciendo busy-poll de `list_agents`, reenviando mensajes a hijos que ya se habían liquidado, y despliegues abandonando `subagent` por `workflow` porque un workflow al menos devuelve algo.

La señal ya existía. `subagent/end` lleva `stopReason` y `lastAssistantMessage` desde que se incluyeron las Activations continuables. Lo que faltaba era cualquier consumidor que la convirtiera en contexto que el modelo del padre pudiera ver.

## Decisión

El gestor de continuación entrega la cuenta por sí mismo, desde dentro de la transacción de disposal que termina la Activation.

Cuando una Activation residente se liquida, `notifySettlement()` resuelve el padre directo durable del hijo y le envía un mensaje en rol de usuario: el resultado de la epoch como una frase sobre la que el padre puede actuar, seguida del contenido final de assistant del hijo, o una declaración de que no produjo ninguno. La entrega es incondicional para cada hijo cuyo id un llamador haya recibido realmente. No consulta si el hijo reportó y no mantiene ninguna contabilidad que pudiera hacer condicional la promesa — esa incondicionalidad es lo que permite a `tool-subagent` prometer un aviso de runtime que contiene el resultado y cualquier mensaje final de assistant. Una materialización revertida antes de su primer mensaje aceptado permanece en silencio, porque al llamador se le dijo que ese hijo no se había establecido.

### Procedencia

El aviso lleva `{ kind: 'subagent-settled', form: 'notice', summary, senderSessionId }`. Deliberadamente no es el tipo existente `subagent-report`. Un report es contenido que el hijo eligió; esto es el runtime declarando qué fue del hijo. Fusionarlos acreditaría al hijo con palabras que nunca escribió, y haría que un log durable no pudiera distinguir «el hijo dijo que había terminado» de «el harness observó que se detuvo». La forma `notice` da también a una UI la presentación colapsada de una línea que este mensaje quiere, donde `relay` lo presentaría como correspondencia.

### Dos reglas de orden, y por qué son del manager

Un listener externo `ctx.on('subagent/end')` parece más desacoplado y está mal. `SubagentRunEndInfo` no nombra a ningún padre, el handle del hijo ya está dispuesto cuando el edge dispara, así que el padre no se puede recuperar de él, y la liberación de propiedad que despierta el propio watcher de liquidación del padre ya se ha ejecutado. El manager mantiene la referencia al padre durante todo el disposal, así que para él no existe ninguno de esos obstáculos.

**El send ocurre antes de `releaseOwnership`.** En ese punto el padre todavía cuenta a este hijo, así que `stateOf(parent)` es `waiting` y el padre es estructuralmente incapaz de ser juzgado liquidado. Entregar después de la liberación, en cambio, compite con un watcher que se reanuda un microtask después, se encuentra sin hijos y en silencio, y dispone un Agent cuyo `cancel()` limpia exactamente la inbox en la que está el aviso. El modo de fallo es un mensaje silenciosamente ausente sin error en ninguna parte.

**Un padre residente lo recibe a través de `admitWaking`.** Registrar el id del mensaje antes del send síncrono es lo que evita que la ventana entre `followup()` y el microtask que lo admite se lea como quietud. Esto no es doble aseguramiento sobre la primera regla: `Agent.status` pliega el mantenimiento de contexto en `idle`, y un send con despertar detrás del mantenimiento solo arma un despertar diferido, así que un padre que compacta su contexto es juzgado en silencio tanto por `status` como por el conjunto de hijos propiedad, en el momento en que llega la liberación.

Ambas reglas están fijadas por tests que fallan cuando se invierte el orden o se elimina la contabilidad.

### Programación

Un padre en reposo recibe un turno posterior ordinario. Un padre ocupado es dirigido a su límite de paso más cercano, porque `Inbox.claim()` toma todo el lote de next-step en un límite: cuatro hijos liquidándose juntos cuestan entonces un paso en lugar de cuatro turnos. Dirigir en lugar de inyectar es deliberado — el despertar es un no-op mientras el driver está en ejecución, y cierra la ventana en la que un driver se retira entre la lectura de estado y el send, lo que dejaría el aviso sin reclamar hasta que algo no relacionado despertara al padre. Esto es una regla de corrección, no una preferencia de despliegue, así que no es un campo `Config`.

Un padre `running` no es dirigible: uno cuyo turno ya está cancelado pero aún no ha salido. `Agent.send()` redirige la entrada con despertar enviada después de la cancelación al siguiente turno, fija el despertar y lo reproduce una vez que el driver cancelado converge — excepto una cancelación de disposal, que nunca fija el despertar y pertenece a la regla de teardown siguiente. El aviso abre por tanto su propio turno sin esperar entrada no relacionada; el costo es un límite de turno redirigido, no el mensaje.

**Un padre cuyo propio teardown comenzó no recibe despertar.** Despertar no es una operación de cola: `Agent.followup()` en un Agent quieto arranca un turno, y `cancel()` en un Agent en reposo es un no-op documentado que no arma contra uno posterior. Cada camino de teardown termina por tanto con un padre vivo, cancelado y todavía registrado — `drainContinuableDescendants()` lo llama el puente ACP entre cancelar sus agents de sesión y disponerlos — así que un aviso sin guarda arranca una petición de modelo real en un Agent a punto de ser destruido, una vez por capa del árbol, porque el aviso de cada capa despierta a su vez a la capa superior. `notifySettlement()` hace la misma pregunta que hace `assertAdmitting()` (¿está cerrada la admisión continuable de este linaje?) e inyecta en su lugar. La inyección no es un buzón durable — Riesgos aceptados registra lo que luego le hace el propio disposal del padre — pero es el único send que llega a un padre que todavía lee su inbox sin armar un turno en uno que no lo hace, y no se pierde nada que el despertar habría entregado: el turno que un despertar arrancó era él mismo dispuesto a mitad de vuelo.

La entrega nunca bloquea ni hace fallar el teardown. Un send rechazado se registra y se descarta, porque retener a un hijo para reintentar un aviso fijaría a todo su ancestro en `waiting` para siempre, y un padre que ha salido del registro es un resultado ordinario, no un error.

### El log de la propia epoch es toda la cuenta

`epochStopReason()` lee el resultado de la epoch de su propio log, porque que el teardown tenga éxito no dice nada sobre si el modelo falló, alcanzó su techo o fue detenido. Leer solo los turnos lo hizo mal dos veces, con la misma forma en ambas: un turno detenido antes de su primer paso deja un `turn/end` indistinguible de los turnos no-op balanceados que producen un rechazo o un reclamo vaciado, así que el filtro que saltaba esos también saltaba los finales reales y respondía con la finalización limpia del turno anterior. El checkpoint de durabilidad (`dsh-session-checkpoint-policy`, en cada profile incluido) y el assembly de prompts corren ambos en ese límite y ambos propagan, y `Inbox.claim()` ya ha tomado los mensajes para entonces — así que al padre se le decía que un hijo había terminado mientras la entrega que esperaba había sido engullida. Bajo el aviso automático de liquidación anunciado, ese es el único fallo que un padre no puede detectar y no reintentará.

El hecho ausente nunca fue del turno; era de la inbox. `Inbox` registra cada mutación con `removedCount` y marca una cancelación `outcome: 'canceled'`, lo que separa un turno que reclama su entrada de un trabajo descartado sin ejecutarse. `foldConsumedWork()` en `dsh-agent` pliega ambos vocabularios en una respuesta: el turno más reciente que da cuenta del trabajo consumido — ejecutado, o reclamado-y-fallido, detenido o rechazado — y si el trabajo aceptado fue cancelado después sin que se abriera un turno sobre él. Un final `blocked` sobre entrada reclamada es también una cuenta: el rechazo de pre-step que lo produjo — un hook que deniega, un plugin de política — descartó los mensajes que el turno reclamó, así que el aviso dice que el hijo declinó en lugar de terminar. Solo un turno `blocked` que no reclamó nada permanece invisible.

Derivarlo del log en lugar del estado vivo es lo que lo hace completo. Una versión anterior muestreaba la propia Activation del manager inmediatamente antes de cancelar, que solo podía ver las cancelaciones que este manager estaba a punto de realizar: un `interrupt()` de un ancestro, o un plugin en descarga cancelando un agent que rastrea, dejaba la muestra falsa y el aviso diciendo aún `finished`. También dejaba el caso aceptado-pero-nunca-reclamado fijado a nada que un test pudiera distinguir de su ausencia. Un solo pliegue sobre el log cubre a cada emisor, y ambas mitades fallan sus propios tests cuando se eliminan.

La precedencia es del consumidor: un fallo o techo registrado gana sobre una cancelación, porque detener a un hijo que ya había fallado no convierte su fallo en una cancelación. `dsh-agent` es dueño del pliegue porque es dueño del marcador de inbox del que depende la respuesta, y ambos consumidores ya dependen de él — la epoch continuable aquí, y el `readResult()` de un solo uso, que tenía el mismo agujero.

Ambos importan más allá del aviso: `subagent/end` lleva `stopReason` a la UI jsonrpc y al puente de hooks de Claude, que informaba de un hijo destrozado a mitad de turno como `completed`.

### Cobertura de snapshot

Tres escenarios ACP ensamblados cubren el aviso: un hijo que nunca reporta, un hijo que reporta primero y un hijo conducido a través de varios turnos de follow-up. Los tres necesitaron una valla explícita. El aviso llega cuando el teardown del hijo termina, lo que compite con lo que el padre ya esté haciendo, así que cada escenario mantiene al hijo detrás del turno de spawn del padre y luego espera el turno del padre que abre el aviso (`waitForTurnStart` en ese turno, luego `waitForTurnEnd`) antes de que el script continúe. Esperar un turno que la ejecución no está vallada a producir no es cobertura: es un timeout cuando el aviso aterriza en el turno ya en ejecución.

`subagent-continuable` es el que fija un fallo. El último turno de su hijo muere en el checkpoint de durabilidad forzado sin entrar en un paso, así que ese transcript es donde la regla de stop-reason de arriba es visible de extremo a extremo: el aviso dice que el hijo *falló*, lleva el `SECOND_OK` anterior como su último contenido en lugar de como resultado, y el turno de acuse de recibo del propio padre llega al cliente ACP.

Un snapshot headless Loader sin clave cubre el camino visible para el usuario de extremo a extremo. Su padre de reproducción omite `run_in_background` para ejercitar el valor por defecto continuable en segundo plano, nunca llama a `list_agents`, `send_message` ni a las herramientas Task, consume el aviso `subagent-settled` escrito por el manager y produce su respuesta final. El hijo nunca llama a `report`, así que el transcript no puede pasar por el camino cooperativo de report. Una valla Loader solo de tests mantiene el request post-spawn del padre hasta que el aviso real del manager entra en su inbox, eliminando la programación de la plataforma del transcript sin sintetizar el aviso.

El escenario `subagent-report` usa la entrega por defecto de report next-step. Una valla solo de snapshot mantiene al hijo hasta que el turno de spawn del padre termina, luego mantiene al padre en mantenimiento hasta que la liquidación sigue al report. El padre reanudado reclama el report de next-step antes de la liquidación de next-turn encolada. La [decisión de ordenamiento report/liquidación](../bug-fix/2026-08-17-subagent-report-settlement-ordering.es.md) es dueña de este ordenamiento entre estados.

Los textos de negativa e interrupción se fijan verbatim en tests unitarios en lugar de en un transcript reproducido: producirlos necesita un plugin de política que rechace o una cancelación vallada en un límite de paso, que los assemblies sin clave no llevan de otro modo, y los escenarios ensamblados ya fijan de extremo a extremo el propio camino del aviso.

## Alternativas consideradas

**Dar a los hijos continuables un Task.** Un Task es un contrato de un solo uso: un productor, una liquidación, un resultado. Una Activation ejecuta muchos turnos, sobrevive a cualquiera de ellos y puede reanudarse después de terminar. Envolverla en un Task recrea exactamente el desajuste de ciclo de vida que los hijos continuables se introdujeron para eliminar, y haría que un turno pareciera terminal.

**Adjuntar un listener externo `subagent/end`.** Rechazado por tres razones arriba — sin padre en el payload, un handle de hijo dispuesto y un orden que el listener no puede influir. Un listener tendría además que ser estrictamente síncrono para ganar a la liberación, y nada en ese seam lo impone, así que la versión correcta solo sería correcta por accidente.

**Entregar solo cuando el hijo no reportó.** Este era el primer diseño. Necesita contabilidad por Activation, sigue perdiéndose al hijo que reportó progreso y luego murió antes de su resultado, y — de forma decisiva — hace condicional la promesa hacia el padre. «Normalmente se te informa» no es un contrato que una descripción de herramienta pueda declarar, y un modelo que no puede confiar en el aviso hará polling igualmente.

**Hacer configurable la entrega.** Un interruptor de despliegue devolvería el texto orientado al modelo a «normalmente», que es el fallo que este cambio existe para eliminar. Las constantes de protocolo y los invariantes de seguridad permanecen fijos; este es uno de ellos.

**Cambiar `subagent/end` para que lleve el padre, y dejar que un plugin entregue.** Eso amplía un payload publicado para un consumidor dentro del paquete, conserva cada peligro de orden y convierte el canal de retorno en un plugin opcional otra vez. Extender el `ActivationObserver` privado del paquete con `terminal(failure)` mantiene un único cálculo de los hechos terminales y ningún cambio de superficie pública.

**Usar siempre `followup`.** Más simple y uniforme, pero un fan-out de hijos liquidándose juntos costaría un turno de padre por cada uno. El lote de límite de paso ya existe; usarlo es gratis.

## Consecuencias

- El padre de un hijo continuable recibe un mensaje por Activation liquidada. Los despliegues con fan-out añaden por tanto turnos de padre; el steering mantiene un lote simultáneo en un paso.
- `tool-subagent` promete el aviso en su schema porque el canal de retorno es comportamiento del servicio, no un plugin opcional.
- `Activation` lleva `parentSession` y `announced`. El primero existe porque el handle del hijo se dispone antes de la entrega; el segundo es lo que mantiene silenciosa una materialización revertida.
- `foldConsumedWork()` reemplaza a `findLastMessageTurnEnd()` de `dsh-session` y se mueve a `dsh-agent`, que es dueño del marcador de inbox que lee; el camino en proceso de un solo uso pliega la misma respuesta y no clasifica como `completed` a un hijo de un solo uso cortado.
- La cobertura unitaria fija el contrato incondicional, cada razón terminal, la programación en reposo y ocupada, el lote, la regresión de mantenimiento, el orden previo a la liberación, un padre que ya no está y un send rechazado que no debe hacer fallar el teardown.
- Tres escenarios ACP usan una valla de liquidación explícita, y `subagent-report` fija el orden por defecto report-antes-de-liquidación de next-step.
- Un snapshot headless Loader sin clave fija inicio en segundo plano → aviso de liquidación escrito por el manager → respuesta final del padre sin polling ni llamada de `report` del hijo.

### Riesgos aceptados

El aviso se entrega, no se confirma. No hay buzón durable, acuse de recibo ni reintento: un padre que no está vivo lo pierde, y la Session del hijo sigue siendo el único registro durable. Cerrar eso necesita un protocolo de buzón sin conexión con sus propias reglas de direccionamiento, autorización y reproducción.

Un aviso inyectado durante el teardown no lo lee un modelo cuando ese padre se dispone a continuación, que es lo que hace cada llamador de teardown: la cancelación del disposal limpia el mensaje sin reclamar y el log conserva el par insert/cancel como registro. Hacer legible la entrega de teardown tras la reanudación requiere o el buzón sin conexión de arriba o un cambio en el disposal del trabajo pendiente durable. El disposal descarta cada elemento de inbox sin reclamar, incluida la entrada de usuario, así que cambiar ese comportamiento es una decisión del agent del núcleo, no un detalle de entrega de liquidación. Tras la reanudación, el padre puede descubrir al hijo pero no recibe el resultado: `list_agents` informa solo de existencia y estado vivo-o-almacenado — `SubagentListEntry.activity` lo dice — y recuperar el final requiere preguntar al hijo mediante `send_message`.

La atribución de stop-reason es un mejor esfuerzo sobre el vocabulario de splice existente del log, sesgado contra sobredeclarar éxito. `Inbox.remove()` y el `clear()` del teardown escriben splices de cancelación idénticos, así que eliminar un mensaje cuyo contenido sobrevive en otra parte — `agent-instructions` aspirando una actualización de instrucción pendiente, o el propio cancel de la liquidación limpiando una dejada pendiente — puede leerse como trabajo descartado sin ejecutar e informar de un hijo terminado como detenido. Separarlos requiere un vocabulario de eliminación más rico en `dsh-agent`; sin él, la mala lectura es estrecha y se inclina hacia el padre que revisa dos veces a un hijo terminado, nunca hacia confiar en uno sin terminar.

La amplificación de turnos es real para árboles profundos o anchos, y no es configurable por diseño. El lote de límite de paso la limita para liquidación simultánea pero no para hijos que se liquidan por separado.

Los reports y sus avisos de liquidación posteriores se ordenan a través del FIFO de next-step del padre. Las liquidaciones independientes de hijos hermanos conservan su orden real de entrega en lugar de un orden sintético entre hermanos.
