# @deepseek-ai/dsh-llm-deepseek

[English](README.md) | Español

Adaptador de chat-completions de DeepSeek para el seam LLM (modelo de lenguaje de gran tamaño) del harness: `fetch` directo + SSE (Server-Sent Events; enmarcado por `eventsource-parser`) que traduce el formato de cable oficial (fuente de verdad: la documentación de la API — guides/thinking_mode, guides/tool_calls, api/create-chat-completion) al protocolo `StreamChunk`.

Existe una segunda implementación del mismo seam respaldada por una librería en `@deepseek-ai/dsh-llm-pi-ai`. Este paquete es dueño de la ruta de provider `deepseek-official` — deliberadamente distinta del nombre de catálogo `deepseek` de pi-ai, de modo que una composición puede montar ambas vías de DeepSeek una al lado de la otra; registrar otro adaptador para `deepseek-official` sigue lanzando `LlmError('DUPLICATE_ADAPTER')`.

La raíz del paquete expone el contrato de plugin de Cordis y `DeepSeekAdapter`; la serialización de cable, el análisis de SSE y los helpers de traducción de chunks no forman parte de ese contrato de raíz.

## Configuración

```yaml
- id: llm-deepseek
  name: '@deepseek-ai/dsh-llm-deepseek'
  config:
    apiKeyEnv: DEEPSEEK_API_KEY  # default; resolved per request via ctx.credentials, then the environment
    baseURL: https://api.deepseek.com # optional; $DEEPSEEK_BASE_URL then the public API when omitted
    thinking: enabled        # optional; provider default is enabled
    reasoningEffort: high    # optional; off | low | high | max — omitted ⇒ high
    maxTokens: 256000        # optional positive per-request output cap; this is the default
    streamIdleTimeoutMs: 300000 # optional; positive finite Node timer delay; five-minute default
    maxRequestFilesBytes: 134217728 # optional positive integer; 128 MiB raw request-image default
    maxInlineRequestImageBytes: 20971520 # base64 fallback high watermark; 20 MiB default
    maxImagesPerRequest: 600       # provider request image-count limit
    imageOffloadByteQuantum: 67108864 # oldest-image removal advances in 64 MiB steps
    inlineImageOffloadByteQuantum: 10485760 # fallback removal advances in 10 MiB steps
    imageOffloadCountQuantum: 20      # count overflow advances in 20-image steps
    filesApiTimeoutMs: 60000           # per-image Files resolution deadline; one-minute default
    fileExpiresAfterSeconds: 604800   # uploaded image lifetime; 1 hour to 30 days
    fileRefreshMarginSeconds: 3600    # replace ids with less lifetime remaining
    fileQuotaCleanupBatch: 100        # oldest harness-owned files deleted before one quota retry
    retryPolicy:             # optional; omission uses normal mode with five retries
      mode: always           # normal | always
      backoff:
        initialDelayMs: 500
        maxDelayMs: 10000
        jitterRatio: 0.1
    defaultContextWindow: 1000000 # optional positive-integer fallback; this is the default
    models:                  # optional; defaults to V4 Flash, V4 Pro, and V4 Flash Vision Exp
      - id: deepseek-v4-flash
        name: DeepSeek-V4-Flash
      - id: deepseek-v4-flash-vision-exp
        name: DeepSeek-V4-Flash-Vision-Exp
        inputModalities: [text, image]
        imagePixelBudget: 640000
        imageMaxBytes: 1048576
      - id: private-reasoner
        description: Company-hosted reasoning model
        contextWindow: 512000
```

El plugin registra la única ruta de provider `deepseek-official` junto con su `retryPolicy` resuelta; la omisión se resuelve al modo normal con cinco reintentos. Una solicitud la selecciona con `provider: deepseek-official`; su `model` se transmite tal cual como la cadena `model` del cable, de modo que cambiar los modelos de DeepSeek no exige un registro en tiempo de ciclo de vida. Omitir `models` anuncia `deepseek-v4-flash`, `deepseek-v4-pro` y el `deepseek-v4-flash-vision-exp` con capacidad de imagen, cada uno con una ventana de contexto de 1,000,000 tokens; una lista explícita reemplaza esos valores por defecto, mientras que `models: []` no anuncia ninguno. Las entradas de catálogo se exponen a través de `ctx.llm.listModels('deepseek-official')` para clientes como los editores ACP y el selector Web, pero siguen siendo orientativas: los ids de modelo no listados se transmiten sin cambios como rutas solo de texto. Un nombre de entrada omitido toma por defecto su id, y omitir `inputModalities` significa solo `text`.

Una entrada de catálogo con capacidad de imagen declara `inputModalities: [text, image]` y puede fijar `imagePixelBudget`, `imageMaxBytes` o `imageDetail: low`. El valor por defecto ordinario es 640,000 píxeles en total y 1MiB de bytes codificados; el detalle bajo toma por defecto 512 por 512 píxeles en total. El almacén de adjuntos escala con `min(1, sqrt(pixelBudget / (width * height)))` y redondea hacia dentro para mantener el recuento de píxeles en o por debajo del tope duro, de modo que un adjunto normalizado de 2048 por 1024 se convierte en unos 1130 por 565 en lugar de un cuadrado forzado. Los codificadores de solicitud se ejecutan de forma perezosa: las imágenes de pocos colores prueban PNG (solo paleta sin alfa) y después WebP 85 y 80, otras imágenes con alfa prueban WebP 85 y luego 80, y otras imágenes opacas prueban JPEG 85 y luego 80; las dimensiones se reducen solo cuando ambos intentos de calidad superan 1MiB. La generación concurrente de un `variantId` comparte una única transformación. Un llamador puede cancelar su propia espera sin interrumpir a otros que esperan; la transformación se detiene cuando no queda ningún esperador. El adaptador normalmente sube los bytes de solicitud derivados exactos mediante `POST /files` y envía bloques `{type: "file", file_id}`. Una resolución de id de archivo fallida o agotada reconstruye toda la solicitud de chat con esas mismas versiones de solicitud como data URLs base64; una solicitud nunca mezcla ids de archivo e imágenes en línea. Toda imagen retenida va precedida de un texto estable que nombra el id de adjunto completo y las dimensiones reales de la solicitud. Las solicitudes de usuario, resultado de herramienta, agent-loop, compactación y directas de `ctx.llm.stream` usan todas esta proyección. Las rutas solo de texto reciben placeholders de adjunto estables mientras el historial duradero conserva sus referencias de imagen.

`maxRequestFilesBytes` y `maxImagesPerRequest` acotan las versiones de solicitud retenidas en 128MiB y 600 imágenes por defecto. Los cuantos de bytes y de recuento no deben superar sus límites correspondientes. Antes de las lecturas de adjuntos, el adaptador usa el tope de bytes de versión de solicitud de cada ruta como cota superior conservadora y elimina el prefijo sobrante más antiguo; solo los adjuntos normalizados retenidos se leen y se transforman. Las longitudes derivadas exactas se vuelven a comprobar sin restaurar las imágenes omitidas. Cuando se cruza el límite de bytes, el prefijo más antiguo avanza más allá del siguiente límite de 64MiB; 129 imágenes de un megabyte eliminan las 65 más antiguas y retienen 64MiB, y ese prefijo permanece sin cambios hasta que el historial duradero supera 192MiB. El desbordamiento de recuento avanza de forma independiente en pasos de `imageOffloadCountQuantum`. Las imágenes eliminadas se convierten en el placeholder fijo visible para el modelo `[image omitted to keep the request within its image limit; older images are omitted first. If this image is still needed, read its file again when a path is available; otherwise ask the user to attach it again.]`. Esta proyección de marca de agua alta evita cambiar un prefijo de solicitud antiguo después de cada imagen nueva.

El fallback en línea tiene un presupuesto base64 independiente. `maxInlineRequestImageBytes` toma por defecto 20MiB e `inlineImageOffloadByteQuantum` 10MiB, de modo que un historial de 21 cargas base64 de un megabyte elimina las 11 más antiguas y retiene 10MiB. El cálculo usa longitudes expandidas en base64. Las versiones de solicitud preparadas se reutilizan byte a byte; el fallback no vuelve a decodificar ni comprimir una imagen. Las asignaciones correctas creadas antes de que falle una imagen posterior siguen indexadas para solicitudes futuras.

Los ids subidos se indexan bajo `DSH_HOME` por ámbito de endpoint/clave de API y `variantId` de solicitud. La variante cubre el id de adjunto normalizado, la versión de transformación, los presupuestos de píxeles y bytes de la ruta y los parámetros del codificador, de modo que la Files API y el fallback en línea se refieren a los mismos bytes deterministas. Las subidas solicitan una vida útil de siete días por defecto y guardan el `expires_at` del servidor. Una asignación local a la que no le quede más de una hora se reemplaza antes de usarse; el adaptador no recupera cada archivo remoto antes del chat. Si el chat informa de ids de archivo caducados, eliminados, ausentes o inválidos y nombra uno o más ids usados por la solicitud, el adaptador elimina exactamente esas asignaciones. Si el provider identifica un estado de archivo obsoleto sin nombrar un id, elimina todas las asignaciones de archivo usadas por ese intento de chat. Después sube de nuevo las versiones de solicitud afectadas y reintenta el chat una vez. Un segundo rechazo por archivo obsoleto limpia las asignaciones identificadas por esa respuesta y se devuelve sin un tercer intento de chat. Una respuesta de subida sin un objeto de archivo completo, recuento de bytes coincidente y `expires_at` nunca se indexa; por tanto, una solicitud posterior vuelve a subir en lugar de confiar en un estado local incoherente. Un índice de subidas local malformado se trata como una caché vacía y lo reemplaza la siguiente subida correcta. La resolución de archivos, incluidos el acceso al índice local y la subida remota, tiene por defecto un plazo de un minuto por imagen. Por tanto, el plazo de inactividad de stream por defecto de cinco minutos deja tiempo para el fallback en línea; un despliegue puede configurar un plazo de inactividad de stream más corto cuando quiera que ese plazo externo termine la solicitud primero. Cada resolución correcta rearma el perro guardián de inactividad externo. Cualquier fallo de resolución conmuta esa solicitud al modo en línea, mientras que las operaciones explícitas de gestión de archivos públicas siguen informando de sus propios fallos.

La resolución concurrente de un `variantId` con ámbito comparte una única subida de Files con cancelación local al esperador. Un fallo de subida por cuota primero pagina y recoge el número configurado de archivos `dsh-` más antiguos, y después elimina ese conjunto antes de un reintento de subida. `DeepSeekFilesClient.delete`, `DeepSeekFileStore.release` y `releaseAll` exponen la recuperación explícita de espacio remoto. Los límites actuales del provider que representa este paquete son 128MiB por subida de Files, 32MiB por imagen referenciada en chat, 10,000 archivos almacenados y 25GiB por clave de API; la versión de solicitud por defecto de 1MiB permanece por debajo de los dos límites por archivo.

`contextWindow` es opcional por modelo configurado y no se expone a través del catálogo orientativo. `ctx.llm.resolveModelInfo('deepseek-official', model).context` devuelve primero el valor exacto del modelo y después `defaultContextWindow` para una entrada sin capacidad o un id de paso directo no listado. El valor por defecto del adaptador es 1,000,000; los plugins sensibles a la presión obtienen así una capacidad propiedad del despliegue sin tratar el selector de modelos como autoritativo. Registrar otro adaptador para `deepseek-official` lanza `LlmError('DUPLICATE_ADAPTER')`.

`maxTokens` es el tope de salida configurado por el adaptador para las solicitudes de conversación y toma por defecto 256,000. Una entrada de catálogo puede llevar su propio `maxTokens`, que gana para ese modelo; una entrada sin él, y cualquier id de paso directo no listado, se resuelve al valor del perfil, de modo que añadir un tope por modelo cambia un modelo en lugar de la ruta. La resolución de modelo exacto expone el ganador como `defaultMaxTokens`; `LlmRuntime` materializa ese valor en `GenerateOptions.maxTokens` antes de que el agent loop escriba `request/header`, de modo que la solicitud de cable sigue siendo reconstruible. Un valor explícito de solicitud o de `AgentOptions.maxTokens` gana y se serializa como `max_tokens`. El adaptador no recorta este presupuesto de solicitud contra `contextWindow`; los despliegues con un contexto menor o un límite de salida del provider inferior deben configurar un `maxTokens` compatible.

El mismo resultado de modelo exacto expone los esfuerzos ordenados `off`, `low`, `high` y `max` bajo `reasoning` para todo modelo de paso directo cuando la política del despliegue permite el modo de pensamiento. `reasoningEffort` selecciona el valor por defecto del despliegue y retrocede a `high` cuando se omite. `agent/request` puede reemplazarlo en cada paso de conversación; el valor resuelto se registra en `request/header`. `low`, `high` y `max` activan el modo de pensamiento y se serializan como el mismo valor oficial de nivel superior `reasoning_effort`; el `off` propiedad del adaptador serializa en cambio `thinking.type: disabled` y omite `reasoning_effort`. Un valor no admitido falla con `UNSUPPORTED_REASONING_EFFORT` antes de la E/S de red.

`thinking: disabled` es un bloqueo de despliegue que publica solo `off` con `off` como su valor por defecto. Omitir `reasoningEffort` o configurarlo como `off` es válido; configurar `low`, `high` o `max` hace fallar la carga del plugin, y un intento directo por solicitud de activar el modo de pensamiento falla antes de la E/S de red. Una solicitud con `GenerateOptions.purpose: 'session-title'` fuerza también el modo de pensamiento desactivado y omite el esfuerzo ya resuelto, reservando su salida acotada para el texto de título visible sin cambiar los valores por defecto de conversación o compactación.

`streamIdleTimeoutMs` acota cada lectura de provider pendiente, incluido el `fetch` inicial, sin contar el tiempo que el Consumer pasa entre chunks. Los comentarios SSE de DeepSeek y las resoluciones de archivo correctas rearman una lectura pendiente como actividad de transporte, pero nunca se convierten en valores `StreamChunk` ni en eventos del log de sesión. Una única señal de aborto estable llega a la solicitud y al lector de cuerpo durante toda la llamada; la caducidad detiene el transporte y lanza `LlmError('TIMEOUT')`, mientras que un aborto anterior del llamador lanza `LlmError('ABORTED')`. El adaptador normalmente hace una solicitud de chat por llamada a `stream()` y hace una segunda solo para la recuperación de archivos obsoletos. Un fallo de resolución de archivo antes del primer chat envía una solicitud en línea. Si la resolución de reemplazo falla después de una respuesta de archivo obsoleto, la solicitud en línea es el reintento permitido. Registra la política de reintentos configurada como metadatos de provider, y `dsh-llm-retry` ejecuta esa política por separado en los límites de paso de agent duraderos.

## Configuración dinámica (ajustes + credenciales)

Los hechos de conexión no se congelan al cargar. `resolveAdapterOptions` es el único paso de resolución explícito de la config en bruto a hechos validados, y el adaptador los vuelve a leer a través de un thunk **una vez por operación**: la URL base, el catálogo, los valores por defecto de solicitud, las políticas de imágenes y de Files y el presupuesto de inactividad surten efecto en la siguiente solicitud, mientras que un stream en vuelo conserva los hechos con los que empezó. Tres seams opcionales alimentan ese thunk:

- **`ctx.settings`** — el plugin registra el espacio de nombres `llm-deepseek` con este mismo schema `Config` y su entrada de `cordis.yml` como `base` de la composición, de modo que una sección `llm-deepseek:` en el documento de ajustes del usuario anula cualquier campo sin reiniciar. Sin un servicio de ajustes montado, la config de la entrada sola mueve el adaptador, sin cambios. Una instantánea de ajustes en vivo que pasa el schema pero falla un límite más allá del schema (un id de catálogo duplicado, un par pensamiento/esfuerzo roto) conserva los últimos hechos buenos y registra el fallo; la config de la entrada en sí sigue haciendo fallar la carga del plugin.
- **`ctx.credentials`** — la clave de API se resuelve por llamada de stream, desde la *misma* instantánea resuelta que suministra el endpoint. La configuración lleva solo `apiKeyEnv`, nunca una clave literal: la referencia se resuelve a través del seam de credenciales y, sin un seam montado, a través de las capas de entorno de confianza. Como los hechos de credenciales viajan con los hechos de conexión, una instantánea de ajustes que el resolver rechaza no aporta ni su endpoint ni su clave: toda la generación anterior sigue sirviendo. Toda clave resuelta se comprueba en cuanto a formato antes de usarse, de modo que un valor que ninguna cabecera HTTP puede portar se rechaza con `LlmError('INVALID_CREDENTIAL')` nombrando el punto de entrada que falla — nunca ninguna parte de la clave — en lugar de aflorar como un `TypeError` de `fetch` opaco. Una solicitud sin clave en ningún sitio falla con `MISSING_CREDENTIAL` nombrando cada punto de entrada de configuración, mientras la ruta sigue registrada y el catálogo sigue siendo navegable — la incorporación de la primera ejecución es «explora modelos, guarda la clave, vuelve a pedir», sin reinicio en medio.
- **`ctx.attachments`** — las solicitudes de imagen resuelven este servicio en el momento de la solicitud, de modo que el orden de carga de Cordis no congela la disponibilidad opcional de imágenes. La ausencia rechaza la entrada de imagen con `UNSUPPORTED_CONTENT`; las llamadas solo de texto no requieren el servicio.

El único hecho capturado en el registro es la política de reintentos: cuando su valor resuelto cambia, el plugin vuelve a registrar la ruta en su lugar (la misma instancia de adaptador, una sección síncrona), de modo que `ctx.llm.providerRetryPolicy('deepseek-official')` informa siempre de la política actual.

El plugin declara también su ruta en el directorio de providers configurables (`ctx.llm.listConfigurableProviders()`): provider `deepseek-official`, espacio de nombres de ajustes `llm-deepseek`, ruta de ajustes vacía — la sección entera es el perfil. Las superficies de configuración usan esa entrada para ofrecer este adaptador junto a los providers pi-ai latentes.

## Atribución de la aplicación

Toda solicitud de chat y de Files API lleva la cabecera de atribución compartida de `attributionHeaders()` de dsh-llm, la línea base obligatoria de `User-Agent` que identifica al harness (consulta [dsh-llm § Atribución de la aplicación](../llm/README.es.md#app-attribution-attributionts)). Las solicitudes directas de DeepSeek y las solicitudes de gateway compatibles con OpenAI no reciben cabeceras de atribución de aplicación específicas del provider bajo este contrato de adaptador; la atribución de aplicación de OpenRouter queda diferida a un futuro adaptador o modo OpenRouter explícito. Una solicitud cuyo `GenerateOptions.purpose` sea `compaction` (la llamada de resumen auxiliar de dsh-compaction-basic) lleva además `x-deepseek-harness-compact: 1`, de modo que el host puede separar el tráfico de compactación de las solicitudes de conversación.

La identidad de la solicitud de DeepSeek es independiente de la atribución de la aplicación. Tras la resolución de credenciales, toda solicitud de provider lleva `x-deepseek-harness-user-id` con el id anónimo estable de [`@deepseek-ai/dsh-anonymous-user-id`](../../identity/anonymous-user-id/README.es.md); una solicitud que lleva `GenerateOptions.sessionId` envía también ese valor exacto como `x-deepseek-harness-session-id`, mientras que una llamada directa sin sesión omite la cabecera de sesión. Ambas cabeceras van a la `baseURL` resuelta, incluido un gateway configurado, y permanecen fuera del cuerpo de la solicitud y del contenido visible para el modelo.

## Notas sobre el formato de cable

- Solo streaming (`stream_options.include_usage` siempre activado). `usage` puede llegar adjunto al chunk de fin o como chunk final solo de uso — el traductor difiere ambos hasta `[DONE]`, de modo que `usage` siempre precede a `finish` y nada sigue a `finish`.
- El esfuerzo `off` propiedad del adaptador se asigna a `thinking: {type: 'disabled'}` y nunca cruza el cable como `reasoning_effort: 'off'`.
- El primer chunk del modo de pensamiento lleva `reasoning_content: ""` — se maneja (sin bloque de razonamiento espurio).
- **Regla de devolución del razonamiento**: todo turno de asistente que llevó razonamiento serializa `reasoning_content` de vuelta en el historial. El modo de pensamiento lo exige en los turnos de llamada de herramienta; DeepSeek lo ignora en el resto, mientras que un gateway que recodifica la conversación para otro proveedor recupera la firma de pensamiento aguas arriba de ese turno aplicando un hash al texto reproducido.
- Los mensajes de usuario con capacidad de imagen conservan el orden texto/imagen. El contenido del rol de herramienta sigue siendo una cadena; las imágenes consecutivas de resultado de herramienta se agrupan en el mensaje de usuario siguiente con `Attached image(s) from tool result:`.
- Contabilidad de caché: `cacheReadTokens` ← `prompt_cache_hit_tokens` / `prompt_tokens_details.cached_tokens`; DeepSeek no informa de ninguna métrica de escritura de caché.

## Errores

Las respuestas no 2xx lanzan `LlmError` con códigos estables: `AUTH` (401/403), `QUOTA` (una respuesta cuyos detalles del provider identifican cuota, saldo o créditos agotados), `RATE_LIMIT` (otros 429), `CONTEXT_WINDOW_EXCEEDED` (un 400 cuyo código, tipo o mensaje del provider identifica un desbordamiento de contexto), `INVALID_REQUEST` (otros 400 y 413), `SERVER` (5xx), `HTTP_<status>` en el resto. Su `failure` serializable conserva el estado HTTP más un retardo válido positivo de `Retry-After` en segundos/fecha y `x-request-id` / `x-deepseek-request-id` cuando están presentes. Si DeepSeek rechaza una imagen normalizada, el mensaje principal nombra el adjunto o el nombre mostrado, la posición de mensaje e imagen duraderos, el tipo de medio normalizado, la profundidad de 8 bits sRGB/sRGBA, las dimensiones y el mensaje del provider. Con varios candidatos y sin id de archivo en el detalle del provider, lista cada imagen posible en lugar de asignar el fallo a la primera. La respuesta en bruto sigue siendo la `cause` del error; nunca es el único diagnóstico visible para el usuario. Las lecturas de adjuntos conservan su código de fallo de adjunto estable en lugar de convertirse en fallos de transporte. Un fallo de transporte anterior a la respuesta (DNS, conexión rechazada, TLS, proxy) lanza `TRANSPORT` nombrando el endpoint configurado y encadenando el rechazo original como `cause`; los abortos del llamador lanzan `ABORTED`, y la señal de cancelación del loop sigue siendo autoritativa. Las violaciones de protocolo lanzan `STREAM_CLOSED` (sin `[DONE]`) o `MALFORMED_RESPONSE` (carga JSON incorrecta). Los `finish_reason` de cable desconocidos (p. ej. `content_filter`, `insufficient_system_resource`) se convierten en chunks `finish {kind: 'error', failure}`, y un stream completado cuyo fin `stop` (o ausente) no abrió ningún bloque de contenido se convierte en un `finish {kind: 'error'}` con código `EMPTY_RESPONSE` (reintentado por la política por defecto).

## Experiencia del modelo

### Solicitud de DeepSeek

#### Lo que ve el modelo

El modelo DeepSeek seleccionado recibe el prompt de sistema del harness, el historial de mensajes, los schemas de herramientas, las secuencias de parada y la config de llamada. El modelo de visión recibe normalmente las imágenes retenidas de usuario y de resultado de herramienta como referencias de la Files API junto a handles de adjunto estables y las dimensiones de imagen de la solicitud; un fallo de resolución de Files envía en su lugar todas las imágenes retenidas como data URLs en línea. Una imagen antigua que excede el presupuesto se representa con el placeholder documentado. El contenido de razonamiento de un turno de asistente anterior se reenvía tal cual, llame o no ese turno a una herramienta.

#### Efecto de tokens

La tokenización del provider gobierna la entrada exacta de tokens de texto e imagen. La devolución del razonamiento lleva la cadena de pensamiento de cada turno razonado a las solicitudes posteriores, mientras que descartar las imágenes que exceden el presupuesto evita volver a pagar esos tokens; el uso de lectura de caché se informa cuando está disponible.

#### Efecto de KV Cache

Un prefijo ensamblado sin cambios, incluidas las imágenes retenidas y los placeholders codificados de forma determinista, es elegible para la reutilización de caché de DeepSeek, que este adaptador informa en el uso. Un cambio de ruta de modelo o cualquier cambio aguas arriba de prompt, schema, prefijo, historial o presupuesto de imágenes puede impedir la reutilización desde el primer token cambiado; la devolución del razonamiento añade contenido en cada turno razonado.

### Respuesta de DeepSeek

#### Lo que ve el modelo

El razonamiento, el texto y los argumentos de herramienta en cadena en bruto se traducen a chunks del harness para que el loop los registre y ensamble.

#### Efecto de tokens

Los tokens generados siguen el esfuerzo de razonamiento registrado de la solicitud y `maxTokens`; solo los bloques retenidos por el loop afectan a la entrada posterior.

#### Efecto de KV Cache

Los bloques de respuesta retenidos por el loop se añaden a la siguiente solicitud y conservan su prefijo reutilizable anterior; los bloques descartados no tienen efecto de caché posterior. Cambiar el provider o el modelo selecciona un dominio de caché distinto.

## Limitaciones conocidas y trabajo diferido

- **Una lista `models` de ajustes reemplaza la lista de la composición por completo** — la fusión en la capa de ajustes es por campo, y los arrays son un solo campo; la fusión de catálogo por entrada necesitaría una forma con claves.
- **`tool_choice` no está mapeado** — no forma parte del vocabulario central (recorte de MVP, compartido con el gemelo pi-ai).
- **Las solicitudes usan `fetch` en bruto, no `@cordisjs/plugin-http`** — sin configuración compartida de proxy/interceptación; la adopción queda diferida hasta que un segundo adaptador la quiera (`TODO(http)`).
- **Los tipos de bloque de contenido añadidos por plugins se omiten** — los bloques centrales de texto y las imágenes admitidas se serializan, y la salida de herramienta vacía cruza el cable como el literal `(no output)`.
- **Las imágenes son adjuntos duraderos solo de entrada** — las URLs externas directas y la salida de imagen del asistente no se admiten; la entrada de DeepSeek usa normalmente la Files API y usa base64 en línea solo para la recuperación por solicitud.
