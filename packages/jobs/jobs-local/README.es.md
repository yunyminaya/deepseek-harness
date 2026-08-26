# @deepseek-ai/dsh-jobs-local

[English](README.md) | Español

Implementación local al proceso del contrato de registro de [`@deepseek-ai/dsh-jobs`](../jobs/README.es.md): `LocalJobRegistry` mantiene todos los registros en memoria, emite ids `<kind>-N` por tipo y entrega instantáneas frescas, nunca estado vivo. Cárgalo como plugin y se registra como `ctx.jobs`.

## Admisión

`maxConcurrentJobsPerOwner` es un entero seguro positivo y toma por defecto `10`. Antes de invocar a un producer, `start()` cuenta los registros `running` y `stopping` del propietario exacto; todos los trabajos sin propietario comparten una cubeta de servicio separada. El historial terminal no ocupa capacidad, y solo el asentamiento `done` del producer libera el lugar de un trabajo en stopping.

A capacidad completa, `start()` falla antes de la ejecución del producer y de la asignación de ids con un error que nombra el límite y dice al modelo que use `job_kill`, que espere a que el trabajo termine de detenerse y que reintente. El registro no encola, no desaloja ni mantiene un segundo contador mutable.

## Ciclo de vida

Los trabajos pertenecen a su propietario y a su backend, no al fiber de la herramienta productora, así que las recargas de productores y controladores no los detienen. El primer trabajo de un propietario adjunta un único effect esperado al ámbito exacto del `Agent`. La disposición del propietario cancela los trabajos de ese objeto, espera la quiescencia del producer y elimina sus instantáneas; los ids de agent o de sesión reutilizados no pueden redirigir una limpieza antigua.

La disposición del servicio cierra los listeners, cancela todos los trabajos vivos, espera sus registros y desacopla los effects de los ámbitos de propietario supervivientes. Si la cancelación del teardown lanza, el servicio fuerza el fallo del registro y advierte de que el trabajo puede quedar huérfano en lugar de bloquearse. Una cancelación que regresa pero nunca asienta `done` sigue siendo indistinguible de una parada lenta y puede atascar el teardown.

El asentamiento es de primero que gana: el resultado terminal más temprano — asentamiento del producer, un `done` rechazado contenido como `failed` o un fallo forzado del teardown — se registra una vez, libera a quienes esperan y notifica a los listeners una vez con contención por listener. Las esperas pendientes marcan el trabajo como informado antes de que corran los listeners, de modo que los informadores de finalización no dupliquen avisos, y una cancelación de teardown lo marca por la misma razón: nadie leerá un aviso dirigido a un propietario que se está destruyendo. La finalización es lo último que anuncia un asentamiento, después de que el registro se confirme y se publique el cambio del conjunto visible, porque un informador puede abrir un turno de modelo de forma síncrona y todos los demás observadores deben haber visto ya el registro asentado.

Los controladores y los listeners se apilan por el ámbito que los registró, con la forma del registro de herramientas: un registro se archiva en el ámbito de su contexto de registro, y una lectura une la capa global con la cadena de ámbitos del propietario. Un único registro de ámbito de proceso responde así las preguntas por propietario de cada propietario — `start()` rechaza `background jobs unavailable: no job controller serves this agent (load @deepseek-ai/dsh-tool-jobs in its composition)` para un propietario cuya propia composición no adjunta ninguno, por muchos que adjunten otras composiciones, y un asentamiento llega solo a los listeners que registró la composición de su propietario.

## Experiencia del modelo

Indirectamente, a través de los plugins productores y de [`dsh-tool-jobs`](../tool-jobs/README.es.md), que renderizan los ids de trabajo, la salida, el estado, la cancelación y los avisos de finalización.

#### Efecto en la caché KV

Sin invalidación directa; el consumer nombrado es dueño de cualquier cambio en el prefijo de solicitud.

## Limitaciones conocidas y trabajo diferido

- **Los trabajos son locales al proceso** — los registros mueren con el proceso del harness; la ejecución duradera o entre reinicios necesita un backend aparte que implemente el seam.
- **Una cancelación silenciosamente inefectiva puede atascar el teardown y retener capacidad** — si `cancel` regresa sin asentar `done`, el registro no puede distinguirla de una parada lenta; el trabajo conserva una ranura de cubeta durante el resto de la vida del servicio, y solo un throw explícito puede forzar su fallo de forma segura.
