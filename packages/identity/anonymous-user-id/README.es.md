# @deepseek-ai/dsh-anonymous-user-id

[English](README.md) | Español

Identidad anónima compartida para la telemetría de sesión, el acuse de recibo directo de feedback y las solicitudes de provider de DeepSeek. `getOrCreateAnonymousUserId()` devuelve un UUID v4 aleatorio con ámbito sobre un home del harness, persistido como la línea simple `$DSH_HOME/.anonymous-user-id` (`~/.dsh/.anonymous-user-id` cuando `DSH_HOME` no está definido). El backend de OpenTelemetry lo informa como Resource `user.id`; `/feedback` incluye el mismo valor en su acuse de recibo; y `dsh-llm-deepseek` lo envía como `x-deepseek-harness-user-id`, lo que permite a los sistemas receptores correlacionar registros sin identidades generadas de forma independiente.

La identidad nunca se deriva del nombre de host, la dirección de red, el remote de git ni otra fuente identificadora. Eliminar `.anonymous-user-id` restablece la identidad en el siguiente arranque del proceso. Los homes de harness distintos tienen identidades distintas.

## Contrato de almacenamiento

Las lecturas y escrituras son síncronas porque tanto la construcción de telemetría en el arranque como la ejecución directa de comandos necesitan una sola API. El resultado queda memoizado por ruta de archivo resuelta durante toda la vida del proceso. El primer escritor usa creación exclusiva y un perdedor concurrente adopta al ganador persistido; un archivo corrupto se reemplaza. La persistencia es de mejor esfuerzo, así que un home no escribible recibe igualmente un UUID local al proceso en lugar de bloquear la telemetría o el feedback.

## Composición

Este paquete es una biblioteca compartida, no un plugin de Cordis. Los Consumers importan `getOrCreateAnonymousUserId()` directamente. Su compañero de invariante está deliberadamente vacío porque el paquete no posee ningún flujo de eventos ni relación mutable pública que pueda comprobarse sin crear la identidad como efecto secundario. `DSH_TELEMETRY_DISABLED` solo detiene la exportación de telemetría; no suprime el acuse de recibo directo de feedback ni la cabecera de provider de DeepSeek.

## Experiencia del modelo

Ninguna, ya que el identificador llega a DeepSeek solo como metadatos de transporte HTTP ocultos para el modelo y nunca entra en el body de la solicitud, el prompt ni el contenido visible para el modelo.

#### Efecto en la caché KV

Ninguno; la cabecera de transporte no cambia ni los tokens ni el prefijo visible para el modelo.

## Limitaciones conocidas y trabajo diferido

- **Sin recuperación tras la eliminación** — la pérdida acuña una identidad anónima nueva por diseño; recuperarla exigiría material de derivación estable que debilitaría el anonimato.
- **Concurrencia de mejor esfuerzo** — un lector que cae en el estrecho intervalo entre la creación exclusiva de un proceso concurrente y la escritura completada puede usar un UUID en memoria distinto para esa ejecución; los arranques posteriores convergen en el valor persistido.
- **Sin identidad entre homes** — no se pueden correlacionar valores distintos de `$DSH_HOME`.
- **Los gateways de DeepSeek configurados reciben el id** — `dsh-llm-deepseek` envía la cabecera estable a su `baseURL` resuelto, incluidos los overrides de despliegue, con independencia del modo de compartición de telemetría.
