# @deepseek-ai/dsh-pwsh-local

[English](README.md) | Español

Service Provider de PowerShell local para el seam de ejecución de `@deepseek-ai/dsh-shell` sobre el servicio [`@deepseek-ai/dsh-subprocess`](../../subprocess/subprocess/README.es.md): `PwshLocalExecutor` lanza `pwsh -NoLogo -NoProfile -NonInteractive -Command <command>` por llamada como proceso gestionado a través de `ctx.subprocess`, y es dueño de todo lo que tiene forma de PowerShell — resolución del ejecutable, valores por defecto y topes de comandos, clasificación de timeout/cancelación, el entorno de terminal amigable para el modelo y la fusión de stdout/stderr orientada al modelo para las lecturas en segundo plano. La mecánica de grupos (salida acotada con respaldo en spill, limpieza de credenciales, escalada del kill, disposal) es del servicio de subprocesos.

La cadena de comandos viaja como UN elemento argv hacia `-Command`: PowerShell mismo analiza el texto y no existe ningún shell intermedio, así que no hay capa de escape de comillas de shell (el dominio de cadenas `bash -c` no tiene equivalente aquí). Las rutas nativas de Win32 (`C:\...`) pasan sin cambios.

La raíz del paquete exporta el plugin `PwshLocalExecutor` por defecto y con nombre, su `Config`, los helpers puros `resolvePwshPath`/`candidatePwshPaths` y las constantes `ENV_OVERRIDES`/`ENCODING_PREAMBLE` que el ejecutor inyecta en cada spawn.

## Config

```yaml
- id: bash
  name: '@deepseek-ai/dsh-pwsh-local'
  config:
    cwd: C:\path\to\workspace   # default: process.cwd()
    timeoutMs: 120000           # default foreground timeout
    maxTimeoutMs: 600000        # cap for per-call overrides
    maxOutputBytes: 64000       # per-stream in-memory cap; overflow spills to disk
    maxSpillBytes: 67108864     # per-stream full-output spill cap
    graceMs: 3000               # kill escalation and post-exit pipe-drain grace
    pwshPath: C:\Program Files\PowerShell\7\pwsh.exe  # explicit executable; else well-known locations, then PATH
```

## Comportamiento

La contraparte Windows de `dsh-bash-local`, que refleja deliberadamente sus semánticas llamada por llamada:

- **Spawn por llamada, sin estado de shell** — cada llamada es un `pwsh -Command` no interactivo nuevo (determinista; sin archivos de perfil). Las banderas `-NoLogo -NoProfile -NonInteractive` deshabilitan los banners de arranque, la carga de perfiles y los prompts que ensuciarían la salida de la herramienta.
- **La entrada de composición es una capa, no la última palabra** — cuando se compone un provider de ajustes, este ejecutor registra el [namespace `bash`](../shell/README.es.md) de la capacidad con la entrada anterior como base, de modo que una sección de usuario en `settings.yaml` se superpone a ella y el siguiente comando se ejecuta con los nuevos presupuestos. El namespace se comparte con la familia POSIX porque un host compone exactamente un provider de `ctx.shell`; un documento escrito en cualquiera de las dos plataformas sigue resolviendo en la otra. Los valores que el schema no puede juzgar (positivo y finito, el límite del temporizador `graceMs`) se rechazan en la escritura, dejando al ejecutor en ejecución con su última sección buena.
- **Salida UTF-8 fijada** — cada comando se ejecuta con `[Console]::OutputEncoding` y `$OutputEncoding` fijados a UTF-8 primero, de modo que el respaldo de Windows PowerShell 5.1 (o cualquier host cuya página de códigos de consola no sea UTF-8) no pueda estropear la salida no ASCII: el colector de subprocesos decodifica los bytes como UTF-8. La codificación de entrada se deja en el valor por defecto del host; pwsh 7 usa UTF-8 por defecto y no se ve afectado.
- **Resolución del ejecutable** — `resolvePwshPath` prefiere un `pwshPath` explícito; después, en Windows, sondea la ubicación de instalación de PowerShell 7, cada entrada de PATH (instalaciones de Microsoft Store; comillas circundantes eliminadas) y Windows PowerShell 5.1 como último recurso heredado, comprobando cada candidato con una sonda lstat que acepta un archivo real o un punto de reanálisis con forma de enlace (un alias de ejecución de aplicación de Store falla el stat contra la ACL de su destino, pero lstat ve el alias en sí); en el resto, recurre a un `pwsh` desnudo resuelto a través de PATH. La resolución es una función pura de `(configured, env, platform)`; se ejecuta en la construcción y solo de nuevo cuando un `pwshPath` almacenado difiere del que se usó para resolver el ejecutable actual, de modo que un cambio de ajustes no relacionado nunca vuelve a sondear el sistema de archivos.
- **Presupuestos configurados sobre grupos gestionados** — `resolve()` rellena `workdir`/`timeoutMs`/`stdoutMaxBytes` desde la configuración, y cada spawn entrega al servicio topes de bytes explícitos, tope de spill y `graceMs`. La gracia debe ser positiva, finita y no mayor que [`MAX_TIMER_DELAY_MS`](../../util/timeout/README.es.md), para que Node pueda representarla con un temporizador. La terminación del árbol (taskkill en Windows, señales de grupo de procesos en POSIX), la gracia de drenaje de tubería posterior a la salida, la truncación que conserva la cola y los archivos spill acotados son mecánica de [`dsh-subprocess-local`](../../subprocess/subprocess-local/README.es.md). Un `ShellExecRequest.stdoutMaxBytes` en primer plano puede elevar el presupuesto de captura de stdout para un llamante de confianza; stderr y las ejecuciones en segundo plano siguen usando `maxOutputBytes`.
- **Clasificación de timeout y cancelación** — `run()` fusiona su timeout limitado por configuración con la señal del llamante en un único plazo; solo el timeout del propio ejecutor informa `timedOut`, una cancelación ascendente informa `aborted`, y un comando auto-terminado no informa ninguno ([timeout-library Agent Note](../../../.agents/notes/implemented/architecture/2026-07-06-timeout-deadline-library.es.md)). Windows informa la terminación forzada como salida 1 sin señal, así que los hechos con marca de señal (`signal`, estado `killed`) son solo POSIX allí; la clasificación timeout/abort es independiente de la plataforma.
- **Entorno de terminal amigable para el modelo** — `NO_COLOR=1 PAGER=cat GIT_PAGER=cat` (sin `TERM=dumb`: ese es un concepto POSIX; `NO_COLOR` lo respetan los renderizadores modernos de PowerShell) fusionado como env ordinario bajo la limpieza de credenciales del servicio y las reglas del canal `DSH_*`; una entrada explícita del llamante sigue ganando.
- **Procesos en segundo plano** — `start()` devuelve un manejador `ShellProcess` en vivo inmediatamente, no se aplica ningún timeout, y `readOutput()` del manejador fusiona las lecturas de stdout/stderr basadas en offset del servicio en un único delta de secciones marcadas con un cursor consumidor. Un proceso aún en ejecución pertenece al servicio de subprocesos, así que sobrevive a las recargas del ejecutor y muere (matado y unido) con el disposal del servicio. Todo lo que tiene forma de tarea (ids, propiedad, sondeo, avisos) vive en el [runtime `ctx.jobs`](../../jobs/jobs/README.es.md) genérico, con el que la capa de herramientas registra el manejador — este ejecutor nunca ve una sesión ni un registro.

## Experiencia del modelo

Indirectamente, a través de `dsh-tool-pwsh`, que renderiza las colas de stdout/stderr acotadas de este ejecutor, los deltas de procesos en segundo plano (a través del runtime de jobs genérico), las rutas de archivos spill y los fallos de infraestructura.

#### Efecto en la caché KV

Sin invalidación directa; el consumidor con nombre es dueño de cualquier cambio de prefijo de petición.

## Limitaciones conocidas y trabajo diferido

- **Sin confinamiento por sí mismo** — este ejecutor siempre ejecuta comandos con la autoridad del proceso del harness; los despliegues que necesitan confinamiento componen un ejecutor bash con sandbox o una política en su lugar.
- **Sin shell persistente ni PTY** — cada llamada inicia un `pwsh -Command` nuevo.
- **La cadena de comandos es texto de PowerShell** — el dominio de `-Command` no tiene capa de escape de comillas de shell, pero un comando orientado al modelo lo analiza el propio PowerShell, así que los errores de sintaxis de PowerShell son fallos de comando, no fallos de lanzamiento.
- **Una nota de fallo de spawn en segundo plano es de entrega única** — el servicio de subprocesos no almacena en búfer ninguna salida para un proceso que nunca se ejecutó, así que el ejecutor inyecta `spawn failed: …` exactamente en un delta de `readOutput()`; un lector que descarte ese delta no puede recuperarlo.
- **La terminación en Windows no informa señal** — un proceso matado a la fuerza termina como salida 1 con `signal: null`, así que la clasificación de estado basada en señales (`killed` POSIX) no aplica en Windows; las paradas iniciadas por `kill()` siguen marcando `killed` directamente.
- **El preámbulo de codificación precede al comando** — PowerShell exige que las sentencias `param(...)`, `#requires` y `using namespace`/`using assembly` estén al principio mismo de un script, así que un comando cuya primera sentencia sea una de esas no puede ejecutarse bajo el preámbulo de salida UTF-8. Envuelve un script `param(...)` en `& { … }` (un bloque param puede encabezar legalmente un bloque de script); las sentencias `using` y `#requires` no tienen solución dentro del comando (`#requires` es inerte dentro de `-Command` en cualquier posición) — ejecuta esos scripts desde un archivo en su lugar.
- **El stdin no ASCII bajo Windows PowerShell 5.1 puede decodificarse mal** — el preámbulo fija solo la codificación de salida; `[Console]::InputEncoding` permanece en el valor por defecto del host porque fijarlo bajo stdin redirigido lanza una excepción. pwsh 7 usa UTF-8 por defecto y no se ve afectado.

Las salvedades de heurística de limpieza y retención de spill viven con [`dsh-subprocess-local`](../../subprocess/subprocess-local/README.es.md), que es dueño de esa mecánica.
