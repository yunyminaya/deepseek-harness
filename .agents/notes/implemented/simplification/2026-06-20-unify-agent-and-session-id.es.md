# Agent Note: Unificar el id del agente y el id de la sesión

Status: implemented

[English](2026-06-20-unify-agent-and-session-id.md) | Español

## Problema

Un par agente/sesión en vivo necesita una identidad para el enrutamiento del registro, el event sourcing y la persistencia. Dar a la fábrica entradas `agentId` y `sessionId` independientes permitiría emparejamientos que ninguna vía de producción puede usar, a la vez que obligaría a cada consumidor a elegir o traducir entre dos nombres para el mismo ciclo de vida.

ACP usa el mismo valor para ambas identidades. Stdio y los hooks también operan sobre el flujo de eventos de sesión y necesitan el agente en vivo correspondiente directamente; ninguna vía de producción vuelve a unir un objeto de agente en vivo con varias sesiones ni impulsa una sesión a través de varios ids de agente.

El [runtime de alcance de agente](../architecture/2026-07-12-agent-scope-runtime-design.es.md) usa una sola `AgentCreationTransaction` para crear y reanudar, y las entradas de agente/sesión comparten la misma regla de colisión de entrada final. Una segunda identidad no representaría una liveness, un rollback o una quietud separados; solo añadiría API y estado de traducción alrededor de la misma transacción.

La identidad de sesión tiene igualmente un único hogar en `Session.header.id`; `Session.id` es un accessor derivado en lugar de un estado independiente que necesite validación duplicada.

## Decisión

El id de registro de un agente es igual a su id de sesión. `CreateAgentOptions` acepta un único `sessionId` usado para ambas entradas finales del registro; la reanudación registra el agente bajo `resumeSessionId`; la creación de subagentes en proceso usa el id de sesión hijo; y `Session.id` se deriva de `header.id`. Una ejecución ACP remota no tiene par agente/sesión local: conserva un id de ciclo de vida acuñado por el padre, mientras que el id de sesión local a la conexión del servidor hijo permanece privado a las llamadas ACP. La transacción de creación existente, las comprobaciones de colisión de entrada final y la semántica de detach de entrada exacta permanecen; los mapas y campos cuyo único trabajo era traducir entre ids locales desaparecen.

La vía guiada por configuración conserva `agents[].id` como una etiqueta de configuración estable, no una identidad de enrutamiento en vivo. Un arranque normal acuña el id combinado `${label}-session-${randomUUID()}` para que los reinicios durables no colisionen. Una app acoplada puede pre-acuñar y pasar un `sessionId` exacto: el primer uso lo crea, mientras que un remount de AgentLoop con un servicio de persistencia ya presente reanuda el historial materializado bajo esa misma identidad. `resumeSessionId` en cambio exige una identidad persistida existente. Las dos entradas de id exacto son mutuamente excluyentes. Stdio usa la forma reanudar-o-crear para que su agente creado por configuración y su UI compartan una identidad opaca a través de los recargas del loop, en lugar de adivinar a partir de un prefijo. Los logs pueden usar la etiqueta estable mientras que todas las búsquedas en vivo y durables usan el único `SessionId`.

`agent/created` y `agent/disposed` permanecen. Son eventos de publicación de ciclo de vida emparejados, no alias de identidad; cualquier eliminación posterior sin consumidores necesita su propia propuesta tras una búsqueda nueva.

## Alternativas consideradas

**Conservar identidades separadas de enrutamiento y de registro.** Una etiqueta configurada estable más una conversación durable nueva es útil, pero no exige dos identidades en vivo: la etiqueta puede seguir siendo metadatos de configuración/visualización mientras el `SessionId` combinado por ejecución es dueño del enrutamiento y la persistencia. Conservar dos ids preservaría los mapas de traducción y permitiría emparejamientos imposibles sin añadir capacidad de ciclo de vida.

## Verificación

- La creación/reanudación de agentes y la creación de subagentes llevan una sola identidad, y `Session` la almacena en un solo lugar.
- La transacción de creación conserva la cobertura de colisión de entrada final, detach de entrada exacta, rollback y quietud sin estado de ciclo de vida específico de identidad.
- ACP, stdio, hooks, la propiedad de bash, la persistencia y el linaje usan el `SessionId` compartido directamente. El backend de subagentes de ACP acuña su id de ciclo de vida en el namespace del padre porque el id de sesión devuelto por un servidor hijo solo es local al servidor; el puente ACP verifica la propiedad exacta de `Agent` a partir del mapa de sesión directa; y JSON-RPC reenvía solo los eventos de ciclo de vida cuyo flag `local` instantáneo del servicio es true, obtiene el padre delegante del portador de eventos con alcance, y no conserva ninguna caché de identidad o linaje hijo.
- La política reanudar-o-crear guiada por configuración es explícita y está cubierta a través de un reinicio durable.
- Una búsqueda de listeners de producción conservó `agent/created`/`agent/disposed` y su semántica de publicación.

## Consecuencias

Esto descarta los diseños latentes de actor multi-sesión y de traspaso de sesión, y convierte la identidad de sesión elegida por el cliente y persistida en la identidad del registro. Si la identidad de enrutamiento separada se convierte en un requisito real, necesita un diseño de ciclo de vida explícito en lugar de un par no restringido suministrado por el llamador.
