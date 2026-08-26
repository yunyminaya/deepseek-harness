# @deepseek-ai/dsh-bash-local

[English](README.md) | Español

Service Provider local para el seam de ejecutor `@deepseek-ai/dsh-shell` sobre el servicio [`@deepseek-ai/dsh-subprocess`](../../subprocess/subprocess/README.es.md): `LocalBashExecutor` lanza `bash -c <command>` por llamada como un grupo de procesos gestionado a través de `ctx.subprocess`, y es dueño de todo lo que tiene forma de bash — los valores predeterminados y topes de comando, la clasificación de timeout/cancelación, el entorno de terminal amigable para el modelo y la fusión stdout/stderr orientada al modelo para las lecturas en segundo plano. La mecánica de grupos (salida acotada con respaldo en spill, limpieza de credenciales (credential scrub), escalada de kill, disposición) es del servicio de subprocesos.

La raíz del paquete exporta el plugin `LocalBashExecutor` por defecto y con nombre, además de su `Config`.

## Configuración

```yaml
- id: bash
  name: '@deepseek-ai/dsh-bash-local'
  config:
    cwd: /path/to/workspace   # default: process.cwd()
    timeoutMs: 120000          # default foreground timeout
    maxTimeoutMs: 600000       # cap for per-call overrides
    maxOutputBytes: 64000      # per-stream in-memory cap; overflow spills to disk
    maxSpillBytes: 67108864    # per-stream full-output spill cap
    graceMs: 3000              # kill escalation and post-exit pipe-drain grace
```

## Comportamiento

- **Spawn por llamada, sin estado de shell** — cada llamada es un `bash -c` nuevo sin login y sin archivos rc.
- **La entrada de composición es una capa, no la última palabra** — cuando se compone un provider de ajustes, este ejecutor registra el [namespace `bash`](../shell/README.es.md) de la capacidad con la entrada anterior como su base, de modo que una sección de usuario en `settings.yaml` se superpone sobre ella y el siguiente comando se ejecuta con los nuevos presupuestos. Los valores que el schema no puede juzgar (positivos y finitos, el límite de temporizador `graceMs`) se rechazan en la escritura, dejando al ejecutor en marcha con su última sección buena; sin provider, o después de que uno se desmonte, lo que se ejecuta es la entrada de composición.
- **Presupuestos configurados sobre grupos gestionados** — `resolve()` rellena `workdir`/`timeoutMs`/`stdoutMaxBytes` desde la configuración, y cada spawn entrega al servicio topes de bytes explícitos, tope de spill y `graceMs`. La gracia debe ser positiva, finita y no mayor que [`MAX_TIMER_DELAY_MS`](../../util/timeout/README.es.md), para que Node pueda representarla con un único temporizador. Los kills de grupos de procesos, el drenado de pipes posterior a la salida, la retención de cola y los archivos de spill acotados son mecánica de [`dsh-subprocess-local`](../../subprocess/subprocess-local/README.es.md). Un `ShellExecRequest.stdoutMaxBytes` en primer plano puede elevar el presupuesto de captura de stdout para un llamador de confianza; stderr y las ejecuciones en segundo plano siguen usando `maxOutputBytes`.
- **Clasificación de timeout y cancelación** — `run()` fusiona su timeout limitado por configuración con la señal del llamador a través de un único plazo; solo el timeout propio del ejecutor informa `timedOut`, una cancelación aguas arriba informa `aborted`, y un comando que se señaliza a sí mismo no informa ninguno ([Agent Note de la librería de timeout](../../../.agents/notes/implemented/architecture/2026-07-06-timeout-deadline-library.es.md)).
- **Entorno de terminal amigable para el modelo** — `NO_COLOR=1 TERM=dumb PAGER=cat GIT_PAGER=cat` impide que los pagers y el color ANSI estropeen los resultados. Estos valores se fusionan como env ordinario bajo la limpieza de credenciales del servicio y las reglas del canal `DSH_*`; una entrada explícita del llamador sigue ganando. Consulta la [Agent Note de stdin/env](../../../.agents/notes/implemented/architecture/2026-06-30-bash-stdin-env-trusted-plugin-api.es.md) y la [Agent Note del entorno gestionado](../../../.agents/notes/implemented/feature/2026-07-10-agent-session-identity-and-log-location.es.md).
- **Procesos en segundo plano** — `start()` devuelve de inmediato un manejador `ShellProcess` en vivo sin timeout, y `readOutput()` fusiona las lecturas stdout/stderr basadas en offsets en un único delta consumidor, colocando stderr bajo un marcador `[stderr]` cuando está presente. Un proceso en marcha pertenece al servicio de subprocesos, sobrevive a las recargas del ejecutor y se mata y se une (join) en la disposición del servicio. Los ids de jobs, la propiedad, el polling y los avisos pertenecen al runtime genérico [`ctx.jobs`](../../jobs/jobs/README.es.md), con el que la capa de herramientas registra el manejador.

## Model Experience

De forma indirecta, a través de `dsh-tool-bash`, que renderiza las colas acotadas stdout/stderr de este ejecutor, los deltas de procesos en segundo plano, las rutas de archivos de spill y los fallos de infraestructura.

#### Efecto de KV Cache

Sin invalidación directa; el consumer nombrado es dueño de cualquier cambio en el prefijo de petición.

## Limitaciones conocidas y trabajo diferido

- **Sin confinamiento por sí mismo** — este ejecutor siempre ejecuta comandos con la autoridad del proceso del harness; los despliegues que necesiten confinamiento componen [`dsh-bash-sandbox`](../bash-sandbox/README.es.md), mientras que la política por llamada allow/deny/ask pertenece a `tools/pre-execute`.
- **Sin shell persistente ni PTY** — cada llamada arranca un `bash -c` nuevo sin login; la persistencia solo de cwd y las sesiones de terminal interactivas siguen diferidas hasta que un flujo de trabajo real las requiera.
- **Solo POSIX** — el binario `bash` está fijado en el código, y la semántica de grupos del servicio subyacente es POSIX; Windows no está soportado.
- **La nota de fallo de spawn en segundo plano es de entrega única** — el servicio de subprocesos no almacena en buffer ninguna salida para un proceso que nunca llegó a ejecutarse, así que el ejecutor inyecta `spawn failed: …` exactamente en un delta de `readOutput()`; un lector que descarta ese delta no puede recuperarla.

Las salvedades de heurística de limpieza y retención de spill viven con [`dsh-subprocess-local`](../../subprocess/subprocess-local/README.es.md), que es dueño de esa mecánica.
