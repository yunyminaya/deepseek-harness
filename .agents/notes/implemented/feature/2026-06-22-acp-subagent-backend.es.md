# Agent Note: Backend de subagente ACP (delegación fuera de proceso)

Status: implemented

[English](2026-06-22-acp-subagent-backend.md) | Español

## Problema

El seam de subagente ([la Agent Note del seam](2026-06-21-subagent-capability-seam.es.md)) se construyó para que múltiples backends coexistan por nombre en `ctx.subagents`. Los backends en proceso (`-spawn`/`-fork`) corren un hijo como segundo `Agent` sobre el MISMO contexto cordis — barato, pero el hijo comparte proceso, client de modelo y herramientas con el padre. El punto entero del seam era soportar también un hijo FUERA DE PROCESO alcanzado por un protocolo, demostrando que la abstracción generaliza a través de una frontera de proceso. Esta Agent Note añade el primer backend de ese tipo: un client de Agent Client Protocol (ACP).

## Decisión

`@deepseek-ai/dsh-subagent-acp` registra un `SubagentProvider` que corre cada agent hijo en un SUBPROCESO LANZADO, conducido sobre ACP como *client*. Es el gemelo invertido en dirección del bridge del lado servidor existente `@deepseek-ai/dsh-acp` (el *agent* ACP): el bridge RESPONDE `initialize`/`newSession`/`prompt`; este backend LOS LLAMA e IMPLEMENTA los callbacks del `Client` (`sessionUpdate`, `requestPermission`). Apuntar el comando de spawn configurado al ejemplo `acp-agent` hace que el harness hable consigo mismo.

### Proceso fresco por ejecución

Cada `start` lanza un hijo nuevo, corre exactamente una sesión ACP (`initialize` → `newSession` → `prompt`), y `dispose` mata el subproceso y espera su salida. Es el ciclo de vida más simple y refleja la forma en proceso de un-hijo-por-ejecución.

### Stub mínimo de client

El client no anuncia NINGUNA capacidad opcional (ni `fs`, ni `terminal`): el hijo se autoabastece de acceso a archivos y terminal en su propio proceso. Las notificaciones `session/update` se consumen — el backend acumula el texto de `agent_message_chunk` como salida del resultado y descarta el resto (pensamientos, tarjetas de llamadas de herramienta), de modo que solo la respuesta final del hijo aflora. `session/request_permission` se auto-responde con una política configurada (`reject` rechaza cada petición, `allow` aprueba mediante la primera opción con forma de permiso) — ninguna pregunta llega a un humano. Proxyar `fs`/`terminal` de vuelta al padre (un modo de workspace compartido) sigue siendo trabajo futuro, como señalaba la Agent Note del seam.

### Sin capacidades en el arranque

Las `capabilities` del provider son todas `false`. Un hijo fuera de proceso no puede honrar el `maxDepth` del padre (no tiene acceso a `parent.options.subagentDepth`) ni el `toolFilter` (es dueño de su propio registro de herramientas), y el primer corte no implementa `outputSchema`. El servicio rechaza una petición que necesite cualquiera de ellas antes de que corra `start`. El backend inyecta solo `subagents` (no `ctx.agents`); lo ÚNICO que lee de `request.parent` es el cwd de la cabecera de sesión (véase la resolución de workspace más abajo) — ningún contexto de conversación, profundidad o estado de herramientas cruza la frontera de proceso.

### Resolución del cwd del workspace

El directorio de trabajo del hijo es una resolución explícita, nunca el cwd del proceso del harness: el override `cwd` del despliegue cuando está configurado (hecho absoluto contra el directorio de lanzamiento y validado en la carga), si no, el cwd de la cabecera de la sesión padre (validado en el arranque), y un rechazo en voz alta antes de que algo haga spawn cuando ninguno existe. Un proceso servidor ACP atiende sesiones de muchos workspaces, así que `process.cwd()` no puede sustituir al workspace de una sesión — el fallback implícito antiguo corría hijos en el directorio de lanzamiento del servidor. Un candidato debe ser una ruta absoluta que nombre un directorio al que el harness pueda ENTRAR (`X_OK` — `statSync().isDirectory()` por sí solo acepta un directorio con modo 600 con el que spawn fallaría con EACCES), y la misma ruta resuelta se convierte a la vez en el cwd del subproceso y en el workspace de `session/new` de ACP.

### Mapeo de StopReason

ACP `StopReason` → harness `SubagentStopReason`: `end_turn`→`completed`, `max_tokens`→`max-tokens`, `refusal`→`refusal`, `cancelled`→`aborted`, `max_turn_requests`→`error` (no hay equivalente limpio — la tarea no terminó), desconocido→`error`. Un fallo de spawn/transporte/RPC resuelve `error` (o `aborted` si se había pedido una cancelación); `result` nunca rechaza ante un fallo del nivel del hijo, según el contrato del seam.

### Seguridad: entorno hijo depurado

El hijo es un proceso separado, así que hereda un entorno. Las variables ambientales con forma de credencial (`/KEY|PASSWORD|SECRET|TOKEN/i`) NO se reenvían por defecto — los secretos del propio harness padre no deben filtrarse implícitamente en un proceso lanzado (la misma política que aplica el ejecutor de bash). Las credenciales PROPIAS del hijo (necesita una clave de modelo) se suministran EXPLÍCITAMENTE vía `config.env`, en capas DESPUÉS de la depuración, de modo que un `DEEPSEEK_API_KEY` intencional sobrevive mientras que un `AWS_SECRET_ACCESS_KEY` incidental no. El stderr del hijo se hereda al stderr del padre (los diagnósticos afloran de forma natural); un evento `error` a nivel de spawn (p. ej. ENOENT por un comando malo) se captura y se corre contra la conducción ACP, de modo que un comando malo liquida `error` en lugar de tumbar al padre con un error no manejado.

## Pruebas

- **Unit/integración sin clave:** Un subproceso ACP con guion ejercita stdio real para el flujo de prompt/salida, todos los mapeos de stop-reason, la cancelación por señal y por disposal (incluidos pre-abort, la carrera pre-sesión y los casos de tubería rota), ambas políticas de permiso, las actualizaciones que no son mensaje ignoradas, la limpieza ante comando ausente, la recarga del provider y las exportaciones del namespace.
- **Composición de Loader sin clave:** Un cordis.yml de solo prueba arranca la app stdio a través del Loader real con el `cwd` del backend omitido; un modelo con guion delega una vez y el hijo con guion demuestra que corrió — y fue anunciado — en el workspace de la sesión padre (la rama de herencia de cwd de extremo a extremo).
- **e2e con clave:** El backend lanza el ejemplo ACP real; su modelo responde `PONG`, escribe `proof.txt`, y el padre verifica el archivo.
- **Hueco de snapshot:** Cada hijo ACP es un proceso separado con su propia sesión de replay, a diferencia del replay por sesión en proceso. Existe cobertura determinista con servidor mock, mientras `TODO(acp-subagent-replay)` rastrea el replay del padre contra un hijo que reproduce.

## Alternativas consideradas

### Por qué quedarse en el SDK 0.25.1

El backend necesita solo `ClientSideConnection`, `ndJsonStream`, `PROTOCOL_VERSION` y los tipos del protocolo client, todos soportados en 0.25.1. La API fluida de 0.28 exigiría migrar las clases de conexión de client y de servidor a través de toda la capa ACP sin mejorar este backend, así que esa actualización sigue siendo un cambio aparte.

### Por qué no un proceso hijo persistente

La agrupación de procesos persistentes (reutilizar un hijo caliente entre ejecuciones) es una optimización de rendimiento aplazada a trabajo futuro — añade complejidad de ciclo de vida de sesión y de recuperación ante caídas que el primer corte no necesita; que cada `start` lance un hijo fresco refleja la forma en proceso de un-hijo-por-ejecución.

## Consecuencias

Cada ejecución paga un subproceso fresco (spawn + `initialize` + `newSession`). El padre hace aflorar solo la respuesta final del hijo: los pensamientos y tarjetas de llamadas de herramienta de `session/update` se consumen y se descartan, y las peticiones de permiso jamás llegan a un humano — la política configurada las responde. El entorno del hijo viene depurado de credenciales por defecto, así que su propia clave de modelo se suministra explícitamente vía `config.env`.

## Hermanos de provider de producto

Los [providers de Codex app-server y Claude Code Agent SDK](2026-08-04-claude-code-and-codex-subagent-backends.es.md) aplican la misma frontera fuera de proceso de spawn/prompt/liquidación/cancelación como hermanos registrados por nombre. A2A sigue siendo un transporte hermano futuro; el backend ACP demuestra que el seam de subagente soporta esta frontera sin poseer protocolos privados de producto.
