# Agent Note: Sidecar de feedback de mensaje ligado al ciclo de vida

Status: implemented

[English](2026-08-10-message-feedback-sidecar.md) | Español

## Problema

El comando `/feedback` existente registra un evento inmutable `feedback/record` a nivel de Session. Ese evento puede liberar un prefijo de telemetría pendiente bajo `FEEDBACK_ONLY`, así que es la autoridad equivocada para una calificación editable positiva/negativa y una nota opcional adjunta a un mensaje de asistente. El feedback de mensaje necesita semántica independiente de actualización y borrado sin entrar en el log canónico de Session, sin cambiar una proyección, sin alcanzar el contexto del modelo ni consentir implícitamente la telemetría.

Un sidecar indexado solo por `SessionId` puede sobrevivir al ciclo de vida del log que describe cuando un id se recrea con una identidad de cabecera distinta. Una revisión a nivel de Session además hace que ediciones de mensajes no relacionadas entren en conflicto, mientras que la lectura/escritura plana del dominio de almacenamiento no tiene compare-and-swap entre procesos. La disposición de Session es solo desenganche del store vivo, no borrado durable, y el seam actual de persistencia de Session no expone ninguna operación de borrado que pudiera poseer una cascada veraz.

## Decisión

`@deepseek-ai/dsh-message-feedback` posee el servicio `ctx.messageFeedback` y almacena el feedback de mensaje como una fila sidecar del dominio de almacenamiento por Session. El sidecar no es contenido del log de Session ni una proyección de Session. No emite ningún evento `feedback/record` y no realiza traspaso de telemetría; los contratos de feedback de comando y feedback de mensaje siguen siendo independientes.

Toda fila utilizable se liga a la identidad de cabecera de la Session inspeccionada `{createdAt, cwd}`, no solo a su `SessionId`. Un desajuste de ciclo de vida se trata como ausencia: `list` no devuelve elementos, y `put` puede reemplazar la fila obsoleta por una ligada a la identidad actual. Un id reutilizado con distinta identidad de cabecera no puede por tanto heredar feedback obsoleto. Una bifurcación recibe su propia identidad de Session y ninguna copia del sidecar: incluso cuando la semilla de la bifurcación contiene los mismos mensajes de asistente, el feedback permanece adjunto a la Session en la que el humano lo registró.

`put` acepta un objetivo solo cuando `SessionPersistence.inspect()` observa un `assistant/message` no vacío de origen-append con ese `MessageId`. Se rechazan los mensajes de origen-reemplazo, los registros de asistente vacíos solo de uso y los objetivos que no son de asistente. La inspección es la autoridad segura en frío: ni publica ni reanuda un Agent ni confirma reparaciones del log frío solo para validar feedback. Un pre-vuelo de `listSnapshots()` en frío clasifica la ausencia definitiva; el fallo de inspección para una Session catalogada sigue siendo un fallo de infraestructura. Una petición en el estrecho intervalo de desenganche-vivo-a-materialización-de-cabecera puede por tanto devolver `session-not-found`, y el llamante reintenta tras la materialización de retiro.

Antes de que `put` confirme una fila sidecar, coloca el log objetivo tras una barrera de durabilidad. Una Session viva que coincida pasa por el checkpoint canónico de `ctx.sessions.flush`, y luego tanto la vía viva como la fría se leen físicamente desde la secuencia cero a través de `SessionPersistence.readFrom`. La identidad de cabecera y el objetivo de la observación resultante se comprueban de nuevo. Un participante de flush ausente, una identidad cambiada, un objetivo desaparecido o un fallo de lectura física impiden la escritura del sidecar, de modo que un elemento de feedback confirmado nunca precede al mensaje de asistente durable al que referencia.

Cada elemento de mensaje lleva su propia versión opaca más las marcas de tiempo `createdAt` y `updatedAt` asignadas por el Host. `put` compara el `ifVersion` del llamante solo con el elemento abordado, de modo que editar un mensaje no invalida otro. La comparación es estricta incluso cuando el valor deseado ya coincide, impidiendo que una petición obsoleta cruce un ciclo de valores ABA; un conflicto devuelve el elemento actual autoritativo para que los llamantes puedan conciliar sin una segunda lectura. Un no-op de versión coincidente conserva la versión y las marcas de tiempo, mientras que una actualización material conserva `createdAt`, reemplaza la versión y evita que `updatedAt` retroceda. Un borrado de algo ya ausente también tiene éxito. Las versiones son tokens de igualdad, no contadores que los llamantes puedan ordenar o sintetizar.

Una cola de mutaciones por Session envuelve la inspección de ciclo de vida, la lectura del sidecar, la evaluación de conflictos y la escritura de la fila completa. Esto hace seriales las mutaciones de una instancia de servicio y conserva el contrato de compare-and-swap por mensaje dentro de un proceso de Host. La disposición del plugin cierra la admisión, drena el trabajo aceptado de la cola y luego cierra el dominio de almacenamiento. La API subyacente del dominio de almacenamiento no proporciona escritura condicional entre procesos, así que la implementación no reivindica linealizabilidad entre procesos ni protección contra actualizaciones perdidas.

`maxNoteBytes` es una elección de despliegue obligatoria y acota la longitud en bytes UTF-8 de una nota opcional; el bundle de Web Host la fija explícitamente a `8192`. El paquete publica el contrato de Host `messageFeedback.list`, `messageFeedback.put` y `messageFeedback.delete` directamente a través de `TypertRemoteService` y `@Remote`. El montaje agregado de Remote del cliente y la UI siguen poseídos por separado y diferidos; su adaptador posterior sigue siendo un consumidor fino de este contrato de Host.

El servicio no realiza ninguna cascada de borrado falsa. `session/disposed` y `host/session-removed` describen desenganche de la propiedad viva, no borrado durable de Session, y la persistencia de Session no tiene actualmente API de borrado. Las filas sidecar pueden por tanto permanecer tras una eliminación del log fuera de banda; un `{createdAt, cwd}` distinto impide que un huérfano así se convierta en feedback de una Session posterior que reutilice el id.

## Alternativas consideradas

**Añadir ediciones al log de Session y derivar una proyección.** Rechazado porque los metadatos editables de UI se volverían historia canónica adyacente a la conversación, las bifurcaciones la reproducirían y heredarían, el borrado exigiría tombstones, y reutilizar `feedback/record` acoplaría silenciosamente una calificación de mensaje al consentimiento de telemetría.

**Indexar el feedback globalmente por `MessageId`, copiarlo en la bifurcación o usar una revisión de Session.** Rechazado porque los ids de mensaje solo significan dentro de un ciclo de vida de Session, las conversaciones bifurcadas necesitan juicios humanos independientes, y las mutaciones de mensajes no relacionadas no deben crear falsos conflictos.

**Extender `KvTable` con compare-and-swap entre procesos en este cambio.** Rechazado porque los backends enviados del dominio de almacenamiento no exponen ninguna primitiva común de escritura condicional. Una cola local de proceso coincide con la topología soportada de un solo Host; una garantía real entre procesos exige un contrato atómico a nivel de backend y es trabajo aparte.

**Borrar el feedback en la disposición de Session.** Rechazado porque la disposición incluye desenganche ordinario y vías de rollback. Tratarla como borrado durable perdería feedback mientras el log de Session sigue existiendo; la limpieza espera a una autoridad real de borrado de Session.

## Consecuencias

El feedback de mensaje es durablemente local e independientemente editable sin cambiar el historial visible para el modelo ni el comportamiento de telemetría. Los llamantes concurrentes en un Host reciben detección de conflictos por mensaje y resultados seguros de reintento, mientras que los despliegues con múltiples escritores sobre la misma raíz de almacenamiento siguen sin soporte. Una identidad de cabecera distinta trata una fila obsoleta como ausente pero no la reclama; un log clonado que conserve el mismo `{createdAt, cwd}` es indistinguible según este contrato. El contrato Remote de Host está disponible ahora; el ensamblaje de cliente y la UI pueden seguir siendo consumidores finos en lugar de tomar posesión de la persistencia o de la semántica de concurrencia.
