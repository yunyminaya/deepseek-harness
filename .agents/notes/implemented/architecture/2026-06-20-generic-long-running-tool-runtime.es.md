# Agent Note: El runtime de trabajos en segundo plano (`ctx.jobs`) y las herramientas genéricas de control de tareas

Status: implemented

[English](2026-06-20-generic-long-running-tool-runtime.md) | Español

## Problema

El bash en segundo plano combinaba originalmente dos responsabilidades: el executor de bash ejecutaba procesos y además gestionaba ids de trabajo, propiedad, lecturas incrementales, cancelación, listeners de finalización y herramientas de control de cara al modelo. Añadir subagentes en segundo plano exigía el mismo ciclo de vida y contrato de interacción. Implementar ese contrato de forma independiente para cada capacidad de larga ejecución duplicaría el aislamiento, la limpieza, la notificación y el comportamiento del prompt, mientras se enseña al modelo un protocolo distinto de recoger-y-detener para cada productor.

El registro de trabajos, las herramientas de control y los avisos de finalización forman una única capacidad del harness. Bash y los subagentes deberían aportar hooks específicos de ejecución sin poseer el comportamiento genérico de tareas.

## Decisión

El grupo de paquetes `jobs/` posee la semántica de los trabajos en segundo plano:

- `@deepseek-ai/dsh-jobs` registra el trabajo en ejecución como `ctx.jobs` y posee los ids de trabajo, la autorización, las instantáneas, las lecturas, la cancelación, la espera, los listeners de finalización y la limpieza.
- `@deepseek-ai/dsh-tool-jobs` expone `job_output`, `job_list` y `job_kill`, inyecta los avisos de finalización y aporta la guía de prompt del sistema para los trabajos en segundo plano.

Las herramientas de larga ejecución son productores. `dsh-tool-bash` adapta un `ShellProcess` para dar salida incremental y cancelación de procesos; `dsh-tool-subagent` adapta una ejecución hija para dar salida final y dispose de la hija. Los seams de capacidad de bash y subagente siguen siendo independientes de las sesiones y del registro de trabajos.

`JobRegistry` es la Service Definition en `@deepseek-ai/dsh-jobs`; el provider local al proceso es `LocalJobRegistry` en `@deepseek-ai/dsh-jobs-local` (el [Agent Note del contrato de registro de tareas](2026-07-26-job-registry-seam.es.md) registra esa separación).

## Contrato de runtime

Los tipos literales viven en la [página de subsistemas de tareas](../../../../docs/subsystems/jobs.es.md). Un productor llama a `ctx.jobs.start()` con un kind, una etiqueta, un `Agent` propietario opcional, un `outputLimitBytes` positivo opcional y una función `run()`. El runtime completa todo el trabajo de preflight (verificación previa) que puede fallar antes de llamar a `run()` y lo invoca una sola vez. Después de que `run()` devuelve los hooks, el registro se confirma sin otro paso que pueda fallar; un productor no puede iniciar trabajo que carezca de un id de trabajo recolectable.

El provider local al proceso también posee la admisión acotada, cuya justificación se registra en la [decisión de admisión acotada de trabajos en segundo plano](../bug-fix/2026-08-11-bounded-background-job-admission.es.md). Su configuración `maxConcurrentJobsPerOwner`, entero positivo seguro, tiene como predeterminado `10`; `start()` deriva el recuento activo de cada objeto `Agent` exacto a partir de los registros `running` y `stopping`, mientras que toda tarea sin propietario comparte un único bucket del servicio. El rechazo por capacidad ocurre antes de `run()` y de la asignación de id, y la resolución del `done` del productor es el único evento que libera la plaza de una tarea en `stopping`. El provider no pone en cola, no adelanta y no conserva un segundo recuento mutable.

`outputLimitBytes` es política de presentación propiedad del productor, no un buffer del registro. El registro lo valida y lo proyecta sin cambios en `JobSnapshot`; las API de control genéricas aplican el tope a la salida completa de cara al modelo después de añadir sus propios metadatos de estado o aviso. Omitirlo conserva el comportamiento existente del controlador, de modo que el runtime no impone un predeterminado oculto a familias de productores no relacionadas.

Un productor de cara al modelo expone ese id confirmado en su valor de éxito canónico, normalmente `{ kind: 'background', jobId }`; la representación nativa puede conservar prosa legible para humanos. Una llamada en segundo plano abortada de antemano falla en lugar de devolver un no-op, porque no existe ninguna tarea que satisfaga el handle prometido. Una vez que el registro publica el id, la cancelación pertenece al controlador propio de la tarea y al runtime de trabajos: una cancelación posterior de la llamada de herramienta productora no debe matar la tarea publicada. `job_kill`, el dispose del propietario y el desmontaje del servicio solicitan la cancelación; la ejecución en primer plano sigue acoplada al `exec.signal` de la llamada.

Los hooks del productor definen tres responsabilidades:

- `cancel(reason?)` solicita la terminación de forma síncrona, es idempotente y debe hacer que `done` se resuelva.
- `done` nunca rechaza y se resuelve solo después de que el productor haya liberado los recursos de la tarea.
- El `readOutput()` opcional devuelve el siguiente delta de salida consumidor. Omitirlo declara una tarea de salida final cuyo resultado terminal proviene de `JobOutcome.output`.

Los estados son `running`, `stopping`, `completed`, `killed` y `failed`. La información específica del productor, como un código de salida o una razón de detención, pertenece a `detail`; el registro no la interpreta. Los kinds de tarea forman una unión de strings extensible por merge, y los ids de trabajo llevan marca y se generan como `<kind>-N`, con un contador por kind.

El runtime adjunta una continuación a `done`, registra el primer resultado terminal, resuelve a los que esperan e invoca los listeners de finalización con contención de errores por listener. La resolución de primero-que-gana importa durante el desmontaje: si `cancel` lanza una excepción, el runtime marca el registro como fallido a la fuerza y advierte de que el trabajo puede quedar huérfano, en lugar de esperar para siempre una promesa que quizá nunca se resuelva. Un resultado posterior del productor no puede sobrescribir ese diagnóstico ni notificar dos veces. Un `cancel` que regresa sin resolver finalmente `done` sigue bloqueando el desmontaje, porque el runtime no puede distinguirlo de una detención lenta y válida.

Los registros de tareas no son efectos de la fibra de la herramienta productora. Recargar una herramienta o un complemento de controlador no mata, por tanto, el trabajo propiedad de un agente y un backend. El dispose propio del servicio de tareas cancela todas las tareas en vivo y espera a los productores que cumplen el contrato.

## Autorización y ciclo de vida del propietario

Los ids de trabajo son globales al runtime y predecibles, así que el registro autoriza cada acceso. `get`, `read`, `wait` y `kill` aceptan el `Agent` llamante; `list` devuelve solo las tareas visibles para ese llamante. Una tarea con propietario es accesible solo para la sesión propietaria exacta. Las tareas sin propietario están abiertas a llamantes que no son agentes y mueren con el servicio de tareas.

La instantánea almacena el `SessionId` con marca del propietario para la autorización, mientras que las operaciones de ciclo de vida conservan la instancia exacta de `Agent` en vivo. Estas identidades sirven a propósitos distintos: la igualdad de sesión concede el acceso, pero la identidad exacta de objeto selecciona la limpieza y la entrega de finalización. Reutilizar un id de agente o sesión no puede redirigir la limpieza ni los avisos de un ámbito antiguo a un reemplazo.

La primera tarea de un propietario adjunta un efecto asíncrono a `owner.ctx`. El dispose de ámbito de agente cancela las tareas en vivo de ese propietario, espera sus registros terminales y elimina sus instantáneas. Este efecto sobrevive a las recargas del productor y se une a la frontera de quiescencia existente del agente. El servicio de tareas conserva el eliminador del efecto para que la recarga del servicio pueda desacoplar las devoluciones de llamada de los ámbitos de agente aún vivos tras el desmontaje global.

Para los productores que cumplen el contrato, `AgentHandle.dispose()` se resuelve solo después de que el trabajo en segundo plano con propietario se haya detenido. El trabajo destinado a sobrevivir a un agente debe iniciarse sin propietario; sobrevivir a reinicios del runtime exige un diseño aparte de trabajos duraderos.

## API de servicio

`JobRegistry` ofrece:

- `start(spec)` para un registro atómico, con preflight y admitido por el provider.
- `get(id, caller?)` y `list(caller?)` para instantáneas que no consumen.
- `read(id, caller?)` para un delta de stream consumidor o un resultado final idempotente.
- `kill(id, caller?, reason?)` para la cancelación.
- `wait(id, timeoutMs, caller?, signal?)` para la espera terminal acotada.
- `onJobDone(listener)` para la observación con ámbito de efecto, entrega al propietario exacto y contención de listeners.
- `attachController(name)` para la valla de disponibilidad del controlador de tareas.

`wait` devuelve la instantánea terminal cuando la tarea se resuelve o la instantánea en vivo cuando expira su timeout. Abortar una espera cancela solo esa espera. Si la resolución ya ha asignado la entrega terminal al que espera, la instantánea terminal sigue ganando. Los que esperan se dan de baja de forma síncrona al abortar, de modo que una resolución en el mismo tick no pueda suprimir un aviso de finalización en nombre de un lector que no recibe nada.

Un productor cargado sin ningún controlador permitiría a los llamantes iniciar trabajo que no pueden recoger ni detener. `dsh-tool-jobs` llama por tanto a `attachController()` durante toda su vida, y `start()` falla antes de la ejecución del productor cuando no hay ningún controlador adjunto. Esta comprobación ocurre en el arranque y no en la carga del complemento, porque los complementos hermanos pueden activarse de forma concurrente. Los controladores personalizados que no son de modelo pueden adjuntarse sin enseñar nombres de herramientas al registro.

## API de control de cara al modelo

`dsh-tool-jobs` registra tres herramientas independientes del kind con tarjetas de UI genéricas:

- `job_output(job_id, wait?, timeout_ms?)` lee la salida y añade siempre `[status: ...]`. Las tareas de stream devuelven solo la salida desde la lectura anterior; las tareas de salida final devuelven su resultado tras la resolución. Las lecturas no bloquean salvo con `wait: true`, cuyo timeout se predetermina y se limita mediante la configuración del complemento. Un timeout de espera informa del estado aún en ejecución y no detiene la tarea.
- `job_list()` devuelve las tareas visibles para el llamante como `<id> [<kind>] <status> — <label>`, o `(no background jobs)`.
- `job_kill(job_id, reason?)` solicita la cancelación de inmediato. La razón registrada opcional se reenvía al productor. Las tareas terminales informan de su estado existente; un cancel del productor que lanza una excepción hace fallar la llamada y deja la tarea en ejecución.

Las lecturas de stream comparten un cursor consumidor con ámbito de tarea porque el lector previsto es el modelo propietario. Una UI o varios lectores independientes necesitan una API de observación aparte que no consuma; compartir este cursor permitiría que los lectores se consumieran la salida unos a otros.

El prompt del sistema dice al modelo que conserve los ids de trabajo, continúe el trabajo independiente en lugar de hacer busy-polling (sondeo ocupado) o duplicar una tarea en ejecución, recoja las tareas relevantes antes de su respuesta final y mate el trabajo que ya no importa. La finalización entrega un mensaje registrado a la sesión del propietario exacto. A un propietario ocupado se le inyecta; a un propietario inactivo se le despierta, según la política acotada de la que es responsable la [decisión de despertar al propietario inactivo](../feature/2026-08-11-background-job-completion-wakes-an-idle-owner.es.md).

El runtime marca una tarea terminal como `reported` cuando una lectura o una espera la entrega, cuando un esperante en vivo ha reclamado la entrega en la resolución, o cuando el modelo la mata explícitamente. Las tareas `reported` no inyectan avisos de finalización redundantes. Los fallos de los listeners se registran de forma independiente, no detienen a los listeners posteriores y ni los que esperan ni el desmontaje los esperan. Cuando una instantánea lleva `outputLimitBytes`, `dsh-tool-jobs` preserva los límites UTF-8 y reutiliza un marcador de truncado existente del productor en lugar de duplicarlo. Las lecturas reservan los sufijos de estado y conservan la cola de salida; los avisos de finalización reservan el prefijo estable `background job <id>` y la instrucción `job_output` antes de truncar kind, label, status, detail o el propio marcador de truncado variables, de modo que el tope mínimo de PTY siga identificando la tarea que hay que recoger. El controlador de trabajos resuelve el tope del productor visible para el llamante en un listener de pre-ejecución antepuesto, antes de que la política pueda denegar o cortocircuitar el despacho, y luego lo aplica mediante la devolución de llamada `finalizeContent` de última milla de las definiciones de tarea, de modo que los errores de herramienta normalizados, los fallos de pipelines exteriores y los resultados de política de texto único no puedan escapar del límite; los resultados de política deliberadamente estructurados en varios bloques conservan que la política posea su forma y su tamaño.

## Adhesión del productor (opt-in)

Cada productor decide si su schema expone `run_in_background` mediante configuración con predeterminados. `dsh-tool-bash`, `dsh-tool-terminal` y cada instancia de `dsh-tool-subagent` usan `enableRunInBackground`, con predeterminado true. Una instancia deshabilitada omite el parámetro y además rechaza en la ejecución un argumento de segundo plano forzado, porque el validador genérico de argumentos permite claves no declaradas. La omisión en el schema anuncia la capacidad; la comprobación de ejecución la hace cumplir.

`ctx.jobs` no reescribe los schemas de los productores. Un bundle reenvía configuración solo para los productores que posee. Si una llamada en segundo plano llega a `start()` sin un controlador adjunto, la valla del runtime falla antes de la ejecución.

## Integraciones de productores

El seam de bash expone `resolve`, `run` y `start`. `start(spec)` devuelve un `ShellProcess` con lecturas incrementales, cancelación, hechos de salida y una promesa de quiescencia que no rechaza. El executor local conserva los handles en vivo solo para que su propio dispose pueda matar y unirse a los procesos. Los llamantes en primer plano siguen usando `resolve` y `run` directamente.

Para bash en segundo plano, `dsh-tool-bash` registra al agente llamante como propietario. Sus hooks mapean `kill()` a la cancelación, `done` a un `JobOutcome` completado o matado, y `readOutput()` a la salida incremental acotada del proceso más los avisos de spill y de sandbox. Las herramientas genéricas de tareas poseen los ids, las líneas de estado, la enumeración, la espera y los avisos de finalización.

Para los subagentes en segundo plano, `dsh-tool-subagent` crea un `AbortController` propiedad de la tarea y comienza el arranque del provider dentro del iniciador de tareas. La cancelación aborta la misma señal antes o después de la publicación del provider. `done` espera tanto el resultado de la hija como su dispose, mapea la salida completada a un resultado final, mapea el abort a `killed`, y mapea otras razones de detención o fallos de infraestructura a `failed`. El historial intermedio de la hija permanece en la sesión hija y no se expone mediante `readOutput()`.

## Alternativas consideradas

### Herramientas de control por capacidad

Unas herramientas de salida/detención separadas para bash y subagente duplican ids, aislamiento, limpieza, notificación y guía, mientras aumentan la carga de schema y protocolo del modelo. Un solo runtime mantiene el comportamiento específico de ejecución en los productores sin clonar el ciclo de vida de las tareas.

### Un backend abstracto inmediato de runtime de tareas

El contrato actual de `JobStart.run()` pasa devoluciones de llamada en proceso y objetos `Agent` exactos. Un backend duradero cambia la identidad, el reinicio, la propiedad y la semántica de observación, así que en el momento de su introducción el registro siguió siendo un servicio concreto en lugar de congelar la frontera equivocada. El [Agent Note del contrato de registro de tareas](2026-07-26-job-registry-seam.es.md) separó más tarde el contrato de la implementación local al proceso sin cambiar esas semánticas en proceso.

### Eventos de autorización o limpieza propiedad del Consumer

Las comprobaciones propiedad del Consumer invitan a un aislamiento inconsistente o ausente en cada controlador nuevo. Un evento de limpieza en difusión obliga a cada listener a filtrar a cada agente y no ofrece ningún eliminador de registro. La autorización central más un efecto con ámbito de propietario da a cada Consumer la misma valla y un hook de ciclo de vida esperado y eliminable.

### Salida bloqueante o una herramienta de espera separada

Bloquear por predeterminado serializaría al padre mientras corre el trabajo en segundo plano. Esperar sin leer añadiría otra llamada de modelo y otro schema sin devolver información útil. `job_output(wait: true)` hace explícito el bloqueo y lo combina con la entrega del resultado.

La espera usa las primitivas compartidas de plazo, pero no la política genérica de timeout de herramientas. Un timeout de espera es una observación correcta que devuelve `[status: running]`; la política genérica lo sustituiría por un error de timeout. Ningún timeout de llamada de herramienta controla la vida de la tarea después de que se haya devuelto un id de trabajo.

### Sumideros de salida propiedad del runtime

Un sumidero push centralizaría el buffer, pero bash ya posee buffers acotados, truncado y archivos de spill detrás de su seam de executor. Tirar de deltas formateados (pull) preserva esa propiedad. Un backend duradero que posea el almacenamiento puede justificar revisar la interfaz del productor.

### Ids aleatorios, promoción o eventos de sesión de ciclo de vida

La autorización, no lo inaveriguable, es la frontera de acceso, y los ids no derivan rutas de archivos; los ids secuenciales con marca mantienen legibles los transcripts. La promoción de primer plano a segundo plano exige un contrato de interacción con el usuario que el SDK no prescribe. Los inicios, las lecturas y los avisos ya se registran como eventos de herramienta y contexto, así que unos eventos de sesión dedicados a las tareas duplicarían hechos visibles para el modelo.

## Pruebas

La cobertura unitaria fija la atomicidad del preflight, los ids por kind, la admisión por propietario exacto y por bucket sin propietario, la ocupación de `stopping`, la liberación terminal, la validación y proyección del límite de salida, los límites de resultados UTF-8 completos, las lecturas de stream y finales, las carreras de timeout y abort en la espera, la cancelación, la resolución de primero-que-gana, la contención de listeners, la supresión de avisos, el aislamiento de propietarios, las instancias de propietario obsoletas, la limpieza de propietarios, el desmontaje del servicio y la valla de ausencia de controlador. Las pruebas de productores cubren el mapeo de procesos de bash, la cancelación del arranque de subagentes, el mapeo terminal y el dispose. La cobertura de instantáneas fija los schemas de las herramientas de control, la guía del prompt y una ruta ACP (Agent Client Protocol) ensamblada en la que el límite configurado rechaza una segunda tarea Bash real en segundo plano con una acción de recuperación `job_kill`.

## Consecuencias

Los comandos de bash y los subagentes comparten un vocabulario de ids, enumeración, formato de avisos, hábito de prompt y conjunto de herramientas de control. Los productores nuevos de larga ejecución implementan hooks de ejecución en lugar de otro registro y otra familia de herramientas. El [recetario de herramientas](../../../../docs/cookbook/adding-a-tool.es.md) apunta a los productores a este contrato.

Un propietario exacto no puede hacer crecer sin límite el trabajo respaldado por `Task` local al proceso, y otro propietario no consume su asignación. Una solicitud de cancelación mantiene ocupada la capacidad hasta que el productor libera de verdad su recurso, de modo que sustituir trabajo que se detiene lentamente no puede superar el presupuesto configurado de recursos en vivo.

El bash en segundo plano con propietario se detiene ahora con su agente en lugar de sobrevivirle. Los procesos en segundo plano no tienen timeout de executor; los llamantes deben matar el trabajo irrelevante o confiar en el dispose del propietario o del servicio. Las lecturas de stream soportan un único lector consumidor, y un productor que regresa de `cancel` sin resolver `done` puede seguir bloqueando el desmontaje. Los trabajos duraderos, los cursores de observación independientes y la promoción a primer plano siguen siendo diseños separados.
