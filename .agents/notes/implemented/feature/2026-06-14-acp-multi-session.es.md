# Agent Note: Multiplexar sesiones ACP concurrentes sobre una conexión

Status: implemented

[English](2026-06-14-acp-multi-session.md) | Español

> Escrita cuando ACP era un bridge de editor, motivada por el modelo de cliente multisesión de Zed. [ACP como protocolo solo de automatización](../simplification/2026-07-23-acp-automation-only-protocol.es.md) eliminó las superficies de editor; la decisión de multiplexación en sí no ha cambiado y esta nota la formula ahora frente al contrato de automatización.

## Problema

Un cliente de automatización ACP puede mantener varias conversaciones vivas sobre un único subproceso del agent. Un bridge de una sola sesión activa obligaría a crear procesos extra e impediría que un controlador padre condujera hijos independientes sobre una misma conexión. La multiplexación introduce riesgos de aislamiento: las respuestas definitivas, la finalización del prompt, la cancelación, las solicitudes de permiso y los ids predecibles de trabajos en segundo plano nunca deben cruzar las fronteras entre sesiones.

## Decisión

El bridge ACP guarda las sesiones vivas en `Map<SessionId, SessionRecord>`. Los callbacks con ámbito de agent usan `ownedRecord`: buscan `agent.session.id` en ese mapa de id a registro y aceptan el registro únicamente cuando este posee exactamente el objeto agent, de modo que un objeto ajeno con el mismo id no pueda reclamar la sesión. Un registro es dueño de su agent, de su disposer exacto y de un prompt en curso opcional con el número de turno durable que finalmente lo resuelve. El encabezado de la sesión es dueño de su cwd; el bridge no mantiene ningún estado paralelo de workspace ni de capacidades del cliente.

Cada callback `session/event` resuelve el registro propietario antes de enviar o resolver nada. Cada sesión permite un único prompt en curso de forma independiente. El prompt captura su propio `turn/start` de mensaje originado por el usuario y se resuelve solo con el `turn/end` correspondiente; los turnos de inyección, los turnos autónomos de plugin o de meta, y un fin tardío de un turno anterior cancelado no pueden resolverlo. `session/cancel` se dirige a un único registro y solo invoca la ruta de cancelación consciente de la cola de ese agent.

La propiedad de los permisos usa la misma comprobación de agent exacto contra el mapa de id a registro. El respondedor de `approval/request` de ACP envía una solicitud de política de máquina de un solo uso solo para la sesión que posee al agent solicitante y delega las solicitudes ajenas o sin llamada. El bridge no tiene estado de elicitación, de selección de configuración ni de otra interacción humana.

Los trabajos de bash en segundo plano llevan un token de propietario opaco igual al id de la sesión propietaria. `job_output` y `job_kill` comparan el token del llamador con la propiedad del trabajo del ejecutor antes de leer o matar; un id de trabajo predecible por sí solo no concede acceso. La propiedad se guarda con la tarea del ejecutor, de modo que una recarga del plugin de herramientas no la borra.

El cierre de la conexión limpia el mapa de sesiones vivas, resuelve cada prompt pendiente como cancelado y dispone todos los `AgentHandle`s en paralelo. Cada handle detiene y espera su loop, vacía la sesión mientras está adjunto, desregistra el agent y elimina la sesión. El cierre está memorizado y lo comparten la desconexión del cliente y la disposición del plugin.

## Alcance del protocolo y del workspace

[ACP v1 permite expresamente varias sesiones concurrentes en una conexión](https://github.com/agentclientprotocol/agent-client-protocol/blob/01beb5fb5eec60e9f516a80d85eb03594bac61e3/docs/get-started/architecture.mdx#L16-L24), y cada sesión nueva lleva su propio `cwd` principal. Este bridge implementa esa multiplexación a nivel de sesión, incluidos distintos workspaces principales según registra la [decisión de cwd por sesión](../architecture/2026-07-02-fs-per-session-cwd.es.md); no crea un subproceso de agent por sesión.

Un proyecto de múltiples raíces dentro de una sesión es una capacidad opcional aparte: ACP define las [raíces efectivas como el `cwd` principal más `additionalDirectories`](https://github.com/agentclientprotocol/agent-client-protocol/blob/01beb5fb5eec60e9f516a80d85eb03594bac61e3/docs/protocol/v1/session-setup.mdx#L313-L367). El bridge de automatización no anuncia capacidad de múltiples raíces y rechaza un `additionalDirectories` no vacío; cada sesión nueva tiene exactamente un workspace, según registra el [contrato del paquete](../../../../packages/acp/acp/README.es.md#protocol-contract).

[El transporte estándar es un subproceso de agent por conexión stdio](https://github.com/agentclientprotocol/agent-client-protocol/blob/01beb5fb5eec60e9f516a80d85eb03594bac61e3/docs/protocol/v1/transports.mdx#L17-L42); por tanto, varias conexiones requieren varios subprocesos o un transporte personalizado, mientras que esta decisión garantiza varias sesiones dentro de una conexión. Dentro de esa conexión, `ctx.sandboxPolicy` resuelve el `cwd` de cada sesión como su propia raíz de `workspace-write`, de modo que los servicios compartidos de bash y de filesystem puedan atender proyectos concurrentes sin conceder escrituras entre proyectos. Esto no añade `additionalDirectories` de ACP; elimina el límite de raíz de todo el proceso en la ruta ya soportada de una raíz principal por sesión.

## Alternativas consideradas

**Una sola sesión viva por conexión** — rechazada. Añade sobrecarga de procesos e impide que un padre programático multiplexe trabajo cancelable de forma independiente.

**Un `ctx.extend()` por sesión** — rechazado. Un contexto hijo no crea por sí mismo una fibra de plugin hijo, de modo que los listeners seguirían perteneciendo a la fibra del bridge. El bridge implementado usa en cambio listeners globales con demultiplexado O(1) explícito y registros propiedad de cada sesión; el ciclo de vida del agent lo posee `AgentHandle`.

**La identidad del objeto agent como propiedad de los trabajos de bash** — rechazada. Un objeto agent reanudado o reemplazado puede representar legítimamente la misma sesión durable. El token de sesión opaco es la identidad transfronteriza que debe sobrevivir a las recargas de plugins.

## Consecuencias

N sesiones pueden devolver respuestas definitivas, mantener un prompt en curso, solicitar permiso y ejecutar trabajos en segundo plano de forma concurrente sin entremezclarse ni resolverse entre sí. Una cancelación en una sesión no afecta a sus vecinas. El bridge paga mapas explícitos y tests de aislamiento, pero no añade un conjunto de listeners por sesión y por tanto evita el abanico de listeners durante conexiones de larga duración.

El bridge no expone ningún método de protocolo para cerrar una sesión viva de forma independiente. Los registros se van juntos en el cierre de la conexión; la navegación y la reanudación pertenecen a las APIs del host, no a este protocolo de automatización.

## Verificación

La suite de múltiples sesiones conduce sesiones concurrentes a través de respuestas definitivas enrutadas, prompts en curso independientes, cancelación dirigida y cierre compartido; las suites de aprobación y de frontera de salida cubren el enrutado de permisos y el rechazo por agent exacto. Los tests de tool-bash demuestran que una sesión no puede leer ni matar el trabajo en segundo plano de otra sesión.
