# @deepseek-ai/dsh-experimental-agent-team

[English](README.md) | Español

Dominio de Agent Teams de raíz implícita. `ctx.agentTeams` posee una plantilla plana Lead/compañero de equipo, un buzón de pares persistente y un DAG de tareas compartido en el log de la sesión del Lead. La [Agent Note de Agent Teams](../../../.agents/notes/implemented/feature/2026-08-05-agent-teams.md) es dueña de las decisiones de coordinación y aislamiento; el [catálogo del subsistema Team](../../../docs/subsystems/agent-team.es.md) registra las formas persistentes literales y la API del servicio.

## Config

```yaml
- id: agent-team
  name: '@deepseek-ai/dsh-experimental-agent-team'
  config:
    maxMembers: 8
    maxTasks: 256
    maxPendingMessagesPerMember: 64
    maxMessageBytes: 65536
    disposalTimeoutMs: 5000
```

Cada límite debe ser un entero seguro positivo. `maxMembers` cuenta cada nombre jamás aprovisionado, incluidos los miembros fallidos, porque los nombres nunca son reutilizables. `maxTasks` cuenta las tareas no eliminadas. El límite del buzón es por destino; el límite de bytes cubre la entrega enmarcada completa, incluidos su id estable y el nombre del remitente. `disposalTimeoutMs` acota la creación admitida, el despacho del buzón y la resolución de Activation propiedad del Team, de modo que la recarga del plugin y el cierre del proceso fallen de forma visible en lugar de esperar para siempre.

El servicio requiere los servicios de Agent, Session, persistencia de sesión y subagente continuable. Una composición sin almacenamiento de sesión persistente no lo activa.

## Identidad del Team y plantilla

Cada raíz de runtime ordinaria es el Lead implícito de un Team cuyo `TeamId` equivale a su `SessionId`; crear un Team es por tanto libre de estado hasta el primer registro de miembro, mensaje o tarea. Un compañero de equipo es un hijo directo con nombre y continuable registrado en la sesión de esa raíz. Los nombres van en minúsculas kebab-case, con un máximo de 64 caracteres, y son inmutables durante toda la vida del Team. Los ids de sesión siguen siendo las identidades de persistencia y autorización.

`spawnTeammate()` primero añade y descarga un miembro en aprovisionamiento, y luego pide al provider de spawn o fork configurado que cree el id de hijo reservado. Un fallo del provider añade un miembro fallido persistente. La admisión exitosa en la bandeja de entrada se descarga en la sesión del hijo antes de que se confirme la arista activa. En la recuperación de la raíz, un registro de aprovisionamiento solo se vuelve activo cuando el hijo persistido de forma independiente tiene descriptores coincidentes de padre directo y continuable, más su mensaje de usuario inicial, ya sea aún pendiente en la bandeja de entrada persistente o ya registrado en el historial; de lo contrario, se vuelve fallido. Si la recuperación gana una carrera de aprovisionamiento en el mismo proceso, la persona creadora acepta el estado terminal coincidente o reporta `TEAM_PROVISIONING_CONFLICT` y drena un hijo que la recuperación ya marcó como fallido. La liberación cierra la admisión, aborta y espera las transacciones admitidas de creación y de despacho del buzón, y luego pide al propietario de la continuación que libere exactamente los hijos directos en vivo de la plantilla y sus descendientes. Los hijos continuables no pertenecientes al Team del Lead permanecen intactos. Los fallos de limpieza hacen que la liberación falle de forma visible. Esto cierra los fallos y las carreras de recarga entre el aprovisionamiento de la raíz y su arista de miembro terminal sin reutilizar un nombre ni conservar una Activation huérfana.

Los hijos nuevos no tienen semilla de historial de padre. Los hijos de fork capturan una vez el prefijo de turnos completados del Lead; el turno de delegación en curso queda excluido. Los registros de Team heredados llevan el `TeamId` de la raíz anterior y se ignoran cuando un fork ordinario se convierte en una raíz de runtime independiente. Los subagentes propiedad del provider que están fuera de la plantilla no se convierten en Leads de Team anidados.

La plantilla reporta las fases persistentes de aprovisionamiento/fallido y el estado en vivo `running`/`idle`. Un compañero de equipo activo pero no residente está `inactive`; una entrega de despertar posterior lo reanuda en frío a través del propietario de la continuación.

## Buzón persistente

`sendMessage()` valida la pertenencia al grupo de pares, añade `team/message/queued` y descarga antes de intentar la entrega. El resultado siempre identifica ese mensaje persistente; `queued` significa que la entrega inmediata se aplazó y no es una instrucción de reenvío. La entrega silenciosa inyecta, descarga y confirma el contexto inmediatamente cuando el destino está en vivo, pero nunca activa un destino inactivo; el mensaje silencioso de un destino inactivo permanece en cola. La entrega de despertar se convierte en el siguiente turno FIFO del destino y lo reanuda en frío cuando hace falta.

El mensaje de destino empieza por `Team message <id> from <name>:` y conserva el mismo id y remitente en `TeamMessageSource`. Una vez que la sesión de destino mantiene de forma persistente esa identidad, ya sea en su bandeja de entrada pendiente o en el historial de mensajes de usuario registrados, el log del Lead añade `team/message/delivered`. Las admisiones inmediatas se serializan por destino en el orden persistente de la cola, y la recuperación despacha los registros en cola menos entregados en el mismo orden. La entrega pliega el estado en vivo y persistido de la bandeja y el historial antes de reintentar, de modo que un fallo entre la aceptación en la bandeja de entrada y la reclamación del modelo no duplica el mensaje. Una descarga exitosa del log del Lead despierta a los llamadores actuales de `waitForChange()`, que entonces vuelven a consultar el estado de autoridad.

La garantía es reintento local al proceso más desduplicación en la sesión de destino, no entrega exactamente una vez entre procesos. Esta release no tiene transacción de buzón compartida entre procesos ni interfaz de línea de tiempo del buzón.

## Tablero de tareas compartido

Las tareas son instantáneas versionadas completas. Cada mutación lleva `expectedRevision`; los llamadores obsoletos reciben `TEAM_TASK_STALE_REVISION` en lugar de sobrescribir un valor más reciente. Cualquier miembro puede crear, leer o reclamar una tarea lista sin propietario. La persona propietaria o el Lead pueden editarla, liberarla, completarla, reabrirla o eliminarla; solo el Lead puede asignar a otro miembro. Los ids numéricos `task-<n>` exigen un sufijo de entero seguro; la creación reporta `TEAM_TASK_LIMIT` en lugar de reutilizar el último id seguro.

Las dependencias deben nombrar tareas actuales no eliminadas y formar un DAG completo sin aristas propias ni duplicadas. Una tarea pendiente está lista solo después de que se complete cada bloqueador. Se rechaza eliminar una tarea que aún tiene un dependiente no eliminado. Las tareas eliminadas permanecen como tumbas para la reproducción y la estabilidad de ids, pero no consumen `maxTasks` ni aparecen en `listTasks()`.

`writeScopes` son prefijos normalizados relativos al workspace. Las vistas avisan cuando se solapan con una tarea en curso, pero nunca bloquean la reclamación ni autorizan escrituras en el sistema de archivos. Son pistas de coordinación, no bloqueos.

`waitForChange()` espera una arista de plantilla, tarea, buzón o estado en vivo que ocurra después del registro, durante entre 10 segundos y una hora; solo reporta si la espera agotó el tiempo y no reproduce un cambio que ya ocurrió. La liberación del runtime suelta las esperas actuales y hace que las esperas posteriores vuelvan de inmediato sin agotar el tiempo. Los llamadores vuelven a leer el estado de autoridad tras el despertar o el agotamiento del tiempo. La cancelación conserva un motivo Error o reporta un motivo no Error a través de `TEAM_WAIT_ABORTED` con inspección estructural en lugar de coerción de objetos. `interrupt()` es solo del Lead y delega en la ruta de interrupción del subagente continuable, que cancela únicamente el turno actual de un compañero de equipo en vivo con `keepInbox`; no libera la propiedad de tareas ni elimina el correo persistente.

El compañero `./invariant` separado reproduce cada evento de Team candidato contra su prefijo de sesión confirmado. La reproducción valida cada payload de Team de versión actual antes de que entre en el estado plegado, y luego rechaza antes de la anexión las transiciones de miembro inválidas, los nombres reutilizados, los ids numéricos de tarea fuera de rango, las revisiones de tarea discontinuas, las dependencias de tarea inválidas, los registros de cola/ack duplicados y los acuses con el destino equivocado. El `seq` y el `time` de los eventos de sesión son dueños del orden y el tiempo en lugar de marcas de tiempo de instantánea duplicadas.

## Experiencia del modelo

### Mensajes entre pares

#### Qué ve el modelo

Cada mensaje de par entregado es un mensaje con rol de usuario. Un primer bloque de texto breve nombra su id de mensaje estable y su remitente; los bloques de contenido originales del remitente van después sin cambios. Los propios registros de plantilla, tarea y buzón son solo de log y nunca entran en el historial de modelo derivado.

#### Efecto de tokens

Cada entrega de par añade el prefijo del remitente más el contenido del mensaje al historial del destino. Las mutaciones de tarea y plantilla no añaden tokens de modelo; su representación orientada al modelo pertenece a los resultados de `@deepseek-ai/dsh-experimental-tool-agent-team`.

#### Efecto de KV Cache

Los mensajes de par se anexan después del prefijo de historial reutilizable del destino. La reanudación en frío reutiliza la conversación persistida antes de anexar un elemento no entregado anteriormente.

## Limitaciones conocidas y trabajo aplazado

- **Un proceso y un único checkout compartido** — los miembros comparten el cwd y ven las ediciones de inmediato; este paquete no ofrece worktree, miembro remoto, merge ni bloqueo de sistema de archivos.
- **Ámbitos de escritura consultivos** — Bash, los formateadores, los generadores de código y los escritores externos directos pueden eludir las comprobaciones de versión del sistema de archivos; los Leads deben coordinar la propiedad y revisar el diff final.
- **Plantilla plana e inmutable** — solo el Lead crea compañeros de equipo directos; no hay Team anidado, renombrado, eliminación ni reutilización de nombres.
- **Sin liberación automática de propiedad** — la inactividad, la interrupción, la salida del proceso y el trabajo fallido no liberan a la persona propietaria de una tarea.
- **El buzón no es exactamente una vez entre procesos** — no se admiten procesos de harness concurrentes sobre un mismo Team.
