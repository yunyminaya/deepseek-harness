# `@deepseek-ai/dsh-headless`
[English](README.md) | Español

El bundle de un solo uso de dsh. [`cordis.patch.yml`](cordis.patch.yml) se aplica directamente sobre [`dsh-base`](../base/README.es.md): aporta la persona de coding y el modo de herramientas, desactiva el HMR, monta el worker de Code Mode como capacidad de ejecución del núcleo e inserta el plugin `headless-runner` de este paquete (config `{task}`, resuelta desde el provider inyectado `headlessStartup`). No monta ningún Host, servidor HTTP, runtime Web ni plugin de navegador.

Después de que el Loader se asienta, el runner lee el [`ctx.agentDefaultModel`](../../core/agent-default-model/README.es.md) compartido, crea un agent (agente) persistido nuevo mediante `ctx.agents`, envía la tarea como un mensaje de usuario ordinario y espera a la quietud. Vacía la Sesión antes de plegar el intervalo de eventos duraderos que posee, escribe el último texto de asistente no vacío en stdout y solicita la salida mediante el hook de host `ctx.appExit` aportado por el lanzador ([`dsh-cmdline`](../../boot/cmdline/README.es.md)) (si el `turn/end` final se completó → 0, si no 1). Una razón `error` terminal además escribe su código y mensaje en stderr; las ejecuciones con éxito mantienen stderr vacío. El proceso no abre ningún puerto de escucha. El texto de la tarea es la línea de comandos de esta app: el provider ordinario `headless-startup` ([`src/startup.ts`](src/startup.ts)) inyecta `ctx.cmdlineArgs` ([`dsh-cmdline`](../../boot/cmdline/README.es.md)), lee el argumento posicional de `dsh --profile headless "task"`, imprime el `--help` de la app y aporta `headlessStartup`; el runner inyecta ese servicio y lee su tarea desde la configuración perezosa. Una tarea ausente o solo de espacios en blanco se rechaza antes de que el runner se active.

## Experiencia de modelo

Ninguna, ya que el runner envía la tarea como un mensaje de usuario ordinario; los prompts y las herramientas pertenecen a las filas de los bundles base y headless.

#### Efecto de KV Cache

Ninguno; el runner no añade nada al prefijo de petición.

## Limitaciones conocidas y trabajo diferido

- **Solo una tarea enviada** — el runner no tiene superficie interactiva de seguimiento; espera a cualquier trabajo que el Agent complete antes de volver al reposo e imprime el último mensaje de asistente no vacío de ese intervalo.
- **`ctx.appExit` es propiedad del lanzador** — arrancar el profile headless fuera del lanzador `dsh` falla en voz alta en la activación hasta que el host aporta la solicitud de salida.
