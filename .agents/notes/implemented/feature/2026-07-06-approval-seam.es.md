# Agent Note: El seam de aprobación — decisiones de permiso de un solo disparo sobre un waterfall de answerers

Status: implemented

[English](2026-07-06-approval-seam.md) | Español

## Problema

Dos llamadores necesitan una decisión cerrada — «¿puede proseguir esta acción concreta?»: la decisión `ask` de `tools/pre-execute` (incluido el `permissionDecision: ask` del puente de hooks de Claude Code) y el reintento de escalada de un solo disparo tras la denegación de la [Agent Note de sandbox](2026-07-06-sandbox.es.md). Un seam compartido les evita inventar vocabularios de resultados, enrutamiento de canales, cancelación y rastros de auditoría separados, al tiempo que garantiza que un despliegue sin answerer jamás pueda conceder una solicitud incontestable. El answerer puede ser un host interactivo o un controlador automatizado.

El problema de enrutamiento es la titularidad: una solicitud de permiso debe llegar al canal que posee al agent que pregunta, fallar en cerrado para los agents que nadie posee y mantenerse fuera de los despliegues que no componen ningún answerer.

## Decisión

Un solo paquete, `dsh-user-approval` (`packages/interaction/user-approval`), es dueño del vocabulario y del servicio `ctx.approval` — el mecanismo. La política — quién responde, y si a una sesión se le pregunta siquiera — vive fuera de él: los answerers son listeners del waterfall de `approval/request` registrados por los plugins dueños de canal (el puente ACP, los adaptadores de host y los scripts de prueba), y un nivel de política por sesión puede decidir antes de que intervenga un canal. Los consumidores (el enrutamiento `ask` de `dsh-tools` y la compuerta de escalada del sandbox) resuelven una pregunta a un resultado cerrado y derivan de él sus propios resultados de herramienta. Esto es deliberadamente un paquete, no los tres del capability seam (véase Alternativas).

### Cómo lo usa un despliegue

Una entrada de `cordis.yml` monta el seam. No cargarla es la exclusión voluntaria que falla en cerrado: los consumidores deniegan las solicitudes incontestables con cero código de aprobación registrado.

```yaml
- id: approval
  name: '@deepseek-ai/dsh-user-approval'
  # config:
  #   policy: never   # deployment default for sessions without an override; 'ask' when omitted
```

La entrada por sí sola aporta el mecanismo, no un canal: sin ningún answerer compuesto, cada `ask` se resuelve `unavailable` y la llamada de herramienta que pregunta deniega — el fail-closed no necesita configuración. Componer la app ACP (`@deepseek-ai/dsh-acp-demo`, como en [el árbol por defecto del ejemplo acp-agent](../../../../examples/acp-agent/README.md)) cierra el círculo: su [puente solo-automatización](../simplification/2026-07-23-acp-automation-only-protocol.es.md) registra un answerer que envía `session/request_permission` al cliente propietario con el id exacto de la llamada de herramienta y las opciones de un solo disparo allow/reject. `policy: never` es la postura sin supervisión — cada `ask` se auto-rechaza de forma determinista, y el valor actual se une a la instantánea del contexto de ejecución. `policy` se valida contra la lista cerrada al cargar el plugin; cualquier otra cosa lanza.

Lo que observa un despliegue compuesto: `allowed-once` deja proseguir exactamente esa llamada; el rechazo, la omisión y la ausencia de canal deniegan con tres razones distintas que el modelo puede diferenciar; una solicitud con éxito dentro de un turno aterriza un par `approval/asked`/`approval/decided` duradero en el log de sesión del agent que pregunta; nada de una concesión persiste más allá de la llamada que preguntó. Una solicitud ociosa o un fallo de anexado de auditoría rechaza en lugar de devolver una decisión sin auditar.

Un `ask` bajo esta composición, del escenario grabado `escalation-approved` del ejemplo de sandbox — el modelo solicita una escalada de sandbox, la compuerta pregunta y el cliente de automatización selecciona Allow once:

```
tool/call        bash {"command": "printf 'escalated\n' > escalated.txt && cat escalated.txt",
                       "sandbox_permissions": "workspace-write",
                       "justification": "the user asked to write escalated.txt in the workspace"}
approval/asked   {"toolName": "bash", "callId": "call_00_…",
                  "reason": "escalate sandbox to workspace-write: the user asked to write escalated.txt in the workspace"}
  → session/request_permission {"toolCall": {"toolCallId": "call_00_…"},
                  "options": [{"optionId": "allow-once", "name": "Allow once", "kind": "allow_once"},
                              {"optionId": "reject-once", "name": "Reject",     "kind": "reject_once"}]}
  ← the client selects "Allow once"
approval/decided {"outcome": "allowed-once"}
tool/result      "escalated" — this one call ran under the wider mode; the grant died with it
```

El gemelo `escalation-rejected` termina con `{"outcome": "rejected"}` en su lugar: no se ejecuta nada, y el resultado del modelo lleva el texto fail-closed verbatim del preguntador (`the user rejected escalating this command to "workspace-write"`). Un `permissionDecision: ask` de un hook viaja por el mismo cable; solo difieren el preguntador y sus textos de denegación (§ Enrutamiento de `ask` en dsh-tools). Sin answerer, la misma solicitud se resuelve `unavailable`.

### Detalle de diseño

#### El seam: separación de mecanismo y política

Tras la validación y un anexado `approval/asked` con éxito, el servicio resuelve el waterfall `approval/request` a `allowed-once`, `rejected`, `cancelled` o `unavailable`. Toma prestadas la identidad y la señal de solo lectura de la solicitud, trata el abort como `cancelled`, contiene los fallos de los answerers y los retornos inválidos como `unavailable`, descarta las respuestas tardías y anexa el evento emparejado `approval/decided`. Los fallos de auditoría previos a la confirmación rechazan; los fallos de los observers posteriores al anexado no pueden deshacer un evento autoritativo. `allowed-once` autoriza solo la acción preguntada, y `request()` rechaza fuera de un turno abierto para que el par de auditoría permanezca dentro del límite de confirmación durable.

Los answerers son listeners del waterfall `approval/request`. Cero listeners caen en `unavailable`; un listener que reconoce la solicitud ocupa el slot de decisión de primera respuesta, mientras que un agent que no la reconoce debe delegar con `next()`. Los listeners se descartan junto con sus fibers, de modo que un canal descargado falla en cerrado. Como el orden de registro entre hermanos no es determinista, un despliegue compone un answerer terminal y reserva `prepend` para las compuertas decide-o-delega.

`ApprovalRequest` lleva el `agent` que pregunta, el `toolName`, el `callId` exacto opcional, el `reason` legible por humanos y la `signal` opcional. Usa la marca `CallId` sin importar `dsh-tools`, que depende de este seam. Los adaptadores de canal correlacionan cualquier estado de llamada más rico por `callId`; la solicitud de aprobación no duplica los argumentos de la herramienta.

#### Enrutamiento de `ask` en dsh-tools

`ToolRuntime.execute()` resuelve `ask` antes del despacho: `allowed-once` procede, mientras que el rechazo, la cancelación y la ausencia de canal producen razones de denegación distintas. El consumo oportunista de `ctx.get('approval')` permite que un servicio ausente o no montado falle en cerrado sin bloquear el fiber del registro. La ejecución sin agent también falla en cerrado porque no tiene ni sesión de auditoría ni dueño de canal.

#### El nivel de política por sesión

El seam también posee la política `'ask' | 'never'` con ámbito de sesión que describe [la Agent Note de sandbox](2026-07-06-sandbox.es.md). La política efectiva se pliega desde los conmutadores registrados sobre el valor por defecto del despliegue. `'never'` se resuelve a `rejected` dentro de `request()` antes de que pueda correr cualquier answerer; `'ask'` despacha y por lo demás cae en `unavailable`. Ambos valores actuales se unen a la instantánea atómica del contexto de ejecución antes de cada solicitud de modelo, de modo que un cambio de política no necesita narración separada; toda solicitud de aprobación sigue registrando el par de auditoría.

#### El answerer ACP

El puente ACP responde solo por un objeto agent exacto que posee su mapa de sesiones. Envía `session/request_permission` con el `callId` existente, anuncia opciones de un solo disparo allow/reject, mapea la cancelación por separado y jamás concede una opción desconocida. Las solicitudes extranjeras o sin llamada delegan; una RPC de cliente fallida se convierte en `unavailable`. Los hooks y `tools/pre-execute` deciden si una llamada pregunta siquiera. Este canal es política de máquina entre un cliente automatizado y su agent, no presentación de ACP.

El answerer enruta a través de la comprobación de titularidad de agent exacto del puente que describe [la Agent Note ACP solo-automatización](../simplification/2026-07-23-acp-automation-only-protocol.es.md), preservando la titularidad de permisos por sesión que exige [la Agent Note multi-sesión](2026-06-14-acp-multi-session.es.md).

#### Auditoría, y lo que ve el modelo

`approval/asked` y `approval/decided` son eventos duraderos de solo registro; el modelo ve solo el resultado ordinario de herramienta derivado del resultado. La finalización con éxito confirma un `decided` por `asked`, incluida la cancelación y el fallo contenido del answerer. Las solicitudes ociosas no anexan ningún evento; un fallo previo a la confirmación rechaza, mientras que el fallo del segundo anexado puede dejar un `asked` ya confirmado sin emparejar.

#### Entidades y dependencias

`dsh-user-approval` depende de Cordis más los contratos de sesión, agent y llamada marcada; `dsh-tools` y `dsh-acp` lo consumen. El ejecutor de sandbox permanece independiente porque `dsh-tool-bash` posee las solicitudes de escalada. El servicio fijo de despacho-y-auditoría sigue siendo un paquete; los answerers reemplazables viven con sus dueños de canal. Las concesiones de capacidad estáticas y las respuestas de permiso del lado hijo de `subagent-acp` siguen siendo asuntos separados.

### Pruebas

Los tests unitarios fijan los resultados, la delegación de primera respuesta, la contención, la cancelación, el enrutamiento con ámbito, el emparejamiento de auditoría, la política `'never'` no eludible, las razones de denegación de herramienta y el mapeo de titularidad/resultado de ACP a través de un puente real con guion.

Las instantáneas registran la escalada de sandbox permitida y rechazada a través de `session/request_permission`, más las contribuciones completas de contexto de ejecución de `'ask'` y `'never'`. Los avisos de permiso sin guion se cancelan y fallan en cerrado.

## Diferido

- **Almacenamiento de la concesión `allow_always`** — honrar una concesión persistente exige diseñar el almacenamiento, la identidad de ámbito (¿llamada? ¿ruta? ¿prefijo? ¿sesión? ¿ventana temporal?) y la revocación; hasta que se diseñe, solo se anuncian las opciones de un solo disparo ([la Agent Note de sandbox](2026-07-06-sandbox.es.md) § Escalada registra la cuestión de ámbito abierta).
- **Un `ask` grabado impulsado por hook a través de un answerer compuesto** — el cable de permisos se graba a través de las ramas de escalada del ejemplo de sandbox. El `hook-cc-pretool-ask` de la matriz de hooks fija la denegación de respaldo sin ApprovalService, mientras que la composición productor-de-hooks-más-answerer permanece en el nivel unitario.
- **Enrutar las aprobaciones de un agent hijo a la sesión del padre** — el hijo de `subagent-acp` se auto-responde sus propias solicitudes de permiso; delegarlas al controlador padre es su propio diseño.

## Alternativas consideradas

- **Un único provider registrado en lugar de listeners de waterfall** — rechazado: una API `registerProvider()` fuerza cada cuestión de composición — pre-filtros de allowlist, decisores de hook externos, respuestas de prueba con guion, una compuerta de política ante un humano — dentro de una sola implementación de provider. El waterfall obtiene composición, ausencia fail-closed y disposición por HMR de maquinaria que el runtime ya tiene; el JSDoc del seam fija la convención del slot de decisión única en lugar de inventar un registro de providers.
- **Una compuerta de permiso en línea `tools/pre-execute` dentro del puente ACP** — rechazado: preguntar por cada llamada propiedad del puente cablea la política de pregunta en el transporte, no puede servir a un segundo preguntador (la escalada de sandbox ocurre después de que empiece la ejecución, sin momento previo a la ejecución) y deja las decisiones `ask` producidas por hooks sin mecanismo compartido.
- **El seam genérico de preguntas de usuario (`ctx.userQuestions`)** — rechazado como mecanismo de aprobación: ambos comparten un esqueleto (enrutar por agent, bloquear por un humano, manejar la ausencia), pero el contrato de la aprobación es más estrecho en todas las dimensiones que importan: un vocabulario cerrado de resultados en lugar de texto libre, un aviso nativo del protocolo adjunto a una llamada de herramienta en lugar de un formulario genérico, ausencia fail-closed obligatoria y eventos de auditoría. La aprobación por tanto no viaja por la vía de elicitación embarcada `packages/interaction/user-questions` / `ask_user_question` — un formulario de elicitación no es un aviso de permiso, y una respuesta de texto libre no es un resultado cerrado; compartir la fontanería de providers sigue abierto si las dos convergen alguna vez.
- **Inyección estática opcional en `dsh-tools`** — rechazado: el tipo `Inject` de cordis vendido no tiene bandera opcional — la forma de objeto mapea nombres de servicio a configuración de interceptación, y un inject declarado bloquea el fiber. `ctx.get('approval')` es el patrón documentado de consumo oportunista (la búsqueda de token propietario de `tool-bash`, la sonda de persistencia del loop), lee la presencia por llamada y degrada correctamente a través de HMR sin maquinaria extra.
- **La división en tres paquetes del capability seam** — rechazado: Service Definition / Service Provider / Consumer encaja a un seam cuyo Service Provider es intercambiable (bash-local vs bash-sandbox). Aquí el cuerpo del servicio es mecanismo fijo y la parte variable son listeners que viven con sus dueños — dividir fabricaría un paquete Service Provider vacío («no dividas preventivamente»).
- **Ofrecer `allow_always` ya** — rechazado: el protocolo puede expresarlo, pero honrarlo exige diseñar el almacenamiento de concesiones, la identidad de ámbito y la revocación (§ Diferido). Anunciar una opción que el harness no puede honrar fabrica concesiones condenadas.

## Consecuencias

El contrato implementado lo fijan las suites de Pruebas:

- `allowed-once` despacha una acción; cualquier otro resultado deniega con una razón distinta, y `'never'` rechaza antes de preguntar.
- Las vías de respuesta ausentes, extranjeras, sin agent, que lanzan, inválidas y desconectadas fallan en cerrado.
- Las solicitudes con éxito enrutan por la titularidad exacta del agent y anexan un par de auditoría reproducible e invisible para el modelo; los fallos ociosos y previos a la confirmación rechazan.
- La titularidad ACP mantiene las decisiones dentro de su sesión, mientras que un despliegue sin el servicio no emite eventos de solicitud ni de auditoría.

Costes y límites aceptados:

- **Dos answerers ansiosos por decidir compiten por el slot.** El orden de listeners entre plugins hermanos no es determinista, así que el seam no puede arbitrar entre answerers terminales rivales — mitigado por convención (un answerer terminal por despliegue; `prepend` solo para compuertas decide-o-delega) en lugar de un mecanismo de prioridad que el bus de eventos no tiene.
- **El ejercicio en producción descansa en una sola composición.** `ask` tiene dos familias de productores — los puentes de hooks a través de `tools/pre-execute` y la escalada de sandbox a través de su propia compuerta — con el cable grabado en la suite de instantáneas del ejemplo de sandbox, así que la cobertura del seam en el mundo real es esa composición hasta que más despliegues la compongan.
- **La titularidad se fija en la identidad del objeto `Agent`.** El answerer resuelve el registro del mapa de sesiones en `agent.session.id` y luego exige que ese registro posea el objeto agent exacto; cada vía actual pasa el mismo objeto por el loop y los seams, pero una frontera futura que clonara o hiciera proxy de los agents haría delegar al puente y fallar en cerrado, y necesitaría un contrato de titularidad distinto.

## FAQ

- **¿Qué ocurre en un despliegue sin ningún answerer (headless, CI)?** Cada `ask` cae por el waterfall vacío a `unavailable` y la llamada de herramienta deniega con la razón «no approval channel is available». El fail-closed es el valor por defecto con cero listeners, no una configuración.
- **¿Puede persistir una concesión — «permitir siempre esto»?** No. `allowed-once` autoriza la única acción preguntada y el servicio no almacena nada entre solicitudes; `allow_always` no se anuncia deliberadamente hasta que se diseñe el almacenamiento de concesiones (§ Diferido).
- **¿Qué ve el modelo de una aprobación?** Solo el resultado de herramienta que el preguntador deriva del resultado — el par de auditoría jamás entra en el transcript. Las tres razones que no conceden son distintas, así que el modelo distingue un «no» humano de un aviso omitido de un canal ausente.
- **¿Quién decide si una llamada pregunta en primer lugar?** Los productores de política: un hook que devuelve `permissionDecision: ask`, cualquier listener de `tools/pre-execute` o la compuerta de escalada del sandbox. El seam y el puente solo enrutan y responden; ninguno inyecta su propio juicio sobre qué merece un aviso.
- **¿Qué ocurre cuando el usuario omite el aviso, o el turno se aborta a mitad del `ask`?** La omisión mapea a `cancelled` con su propio texto de denegación. Una señal ya abortada se resuelve `cancelled` sin despachar; un abort durante el `ask` descarta la respuesta tardía. Cuando ambos anexados de auditoría se confirman, cualquier vía registra un par, nunca dos.
- **¿Y si el cliente responde con una opción que el harness nunca ofreció?** Cualquier selección distinta del `allow_once` ofrecido mapea a `rejected` — un optionId desconocido de un cliente no conforme jamás puede conceder.
- **¿Cómo enrutan las aprobaciones de los subagentes?** No enrutan: la delegación fija a todo hijo en proceso a `'never'` ([decisión de aprobaciones fijadas](2026-08-10-subagent-approval-pinned-never.es.md)), así que cada `ask` de un hijo se resuelve `rejected` antes de cualquier answerer y el hijo lo sabe de antemano a través de su contexto de ejecución. La auto-respuesta del lado hijo de `subagent-acp` es aparte; enrutar los `ask` de un hijo al controlador padre está diferido (§ Diferido).
- **¿Qué cambia realmente `policy: 'never'` en ejecución?** El servicio resuelve cada `ask` de esa sesión a `rejected` antes de despachar cualquier answerer (dentro del servicio, de modo que ningún orden de registro puede eludirlo); la siguiente instantánea atómica del contexto de ejecución declara la política; cada auto-rechazo con éxito registra el par de auditoría.
- **¿Qué ocurre en un hot reload, o cuando un answerer se descarga a mitad de sesión?** Los answerers se descartan con su fiber propietario, así que el siguiente `ask` degrada a `unavailable` en lugar de colgarse en un canal muerto; el remontaje vuelve a registrar el answerer sin estado de puesta al día.
- **¿De dónde saca el cliente el contexto de aprobación?** La solicitud lleva el `callId` exacto y el `reason` legible por humanos del preguntador; los adaptadores de canal pueden correlacionar estado más rico de la llamada de herramienta sin duplicar argumentos en el seam de aprobación.

## Precedentes

Precedentes dentro del repo que este diseño copia o contrasta:

- La compuerta `fs/write-intent` (`packages/fs/fs/`) — las semánticas documentadas de waterfall de slot de decisión de ocupación única (la primera respuesta gana, delegar con `next()`) que reutiliza el contrato de answerer.
- `hook/invoked`/`hook/result` — el precedente de par de auditoría de solo registro que siguen `approval/asked`/`approval/decided`; [la Agent Note de puentes de hooks](2026-06-30-hook-bridges.es.md) embarca `permissionDecision: ask`, el primer productor.
- [La Agent Note de puntos de extensión de intercepción](2026-06-30-interception-extension-points.es.md) — el vocabulario `allow`/`deny`/`ask` de `tools/pre-execute` cuyo `ask` atiende este seam.
- [La Agent Note ACP solo-automatización](../simplification/2026-07-23-acp-automation-only-protocol.es.md) — la comprobación de titularidad de agent exacto contra el mapa de sesiones por la que enruta el answerer; [la Agent Note multi-sesión](2026-06-14-acp-multi-session.es.md) — el bloqueador de titularidad de permisos por sesión que esto implementa.
- El patrón de consumo oportunista `ctx.get()` (la búsqueda de token propietario de `tool-bash`, la sonda de persistencia del loop) — cómo consume `dsh-tools` el seam sin bloquear su fiber en él.
