# @deepseek-ai/dsh-shell

[English](README.md) | Español

El **`ShellExecutor`** (`ctx.shell`) define QUÉ hace un backend bash — ejecutar comandos en primer plano e iniciar procesos en segundo plano — sin decir CÓMO. Los ids de jobs, la propiedad, la recogida, la cancelación y los avisos pertenecen al runtime genérico `ctx.jobs`.

Este paquete es dueño del rol de Service Definition de la capacidad bash, dividido para que cada rol pueda evolucionar (e intercambiarse) de forma independiente:

| Paquete | Rol |
|---|---|
| `@deepseek-ai/dsh-shell` (este) | Service Definition: servicio abstracto + tipos de vocabulario |
| `@deepseek-ai/dsh-bash-local` | Service Provider: subprocesos locales |
| `@deepseek-ai/dsh-bash-sandbox` | Service Provider: la mecánica de `dsh-bash-local` con cada spawn confinado vía [`ctx.sandbox`](../../sandbox/sandbox/), denegaciones informadas como hechos de resultado |
| `@deepseek-ai/dsh-tool-bash` | los schemas de herramienta orientados al modelo sobre `ctx.shell` |

La división es un seam de capacidad estándar ([capability-seams Agent Note](../../../.agents/notes/implemented/architecture/2026-06-13-capability-seams.es.md)): `dsh-bash-sandbox` es un ejecutor con sandbox detrás de la misma Service Definition — el Consumer detecta su capacidad `sandboxMode` y añade campos de escalada sin importar el provider — y un ejecutor contenerizado o remoto encaja de la misma manera.

## API del servicio (`ctx.shell`)

| Miembro | Semántica |
|---|---|
| `run(spec)` | Ejecución en primer plano. Resuelve cuando el comando termina. **Rechaza solo por fallos de infraestructura** (workdir inutilizable, shell ausente, señal pre-abortada); las salidas distintas de cero, los kills por timeout y los kills por aborto resuelven con un `ShellRunResult` descriptivo. |
| `start(spec)` | Ejecución en segundo plano. Devuelve inmediatamente un manejador `ShellProcess` sin tarea; **no se aplica ningún timeout**. El llamante puede adaptarlo a `ctx.jobs`. |
| `sandboxMode` | El hecho de capacidad para la capa de herramientas: el modo por defecto bajo el que confina un ejecutor CON sandbox (`undefined` en la clase base — «este ejecutor no aplica sandbox»). `dsh-tool-bash` lo lee en el registro para anunciar los campos de escalada solo cuando la composición los respeta. |
| `ShellProcess.readOutput()` | **Lectura incremental** de la salida — las lecturas consecutivas nunca re-entregan. Las lecturas que perdieron datos por los límites del búfer marcan `lossy` y apuntan a los archivos spill de flujo completo. |
| `ShellProcess.kill()` | Mata el grupo de procesos. Devuelve `false` cuando ya terminó. |

Las implementaciones subclasifican `ShellExecutor` e implementan los métodos abstractos. El disposal debe matar cada proceso en ejecución y esperar su salida.

`SHELL_SETTINGS_NAMESPACE` (`bash`) se exporta aquí y no desde un provider porque nombra la capacidad, no una implementación. Un host compone exactamente un provider de `ctx.shell` — la capa win32 intercambia las filas POSIX por las pwsh, y montar ambas falla en voz alta por registro de servicio duplicado — de modo que cada provider puede registrar este único namespace con su propio schema y entrada de composición sin que dos lleguen jamás a colisionar, y un `settings.yaml` llevado entre plataformas sigue resolviendo en ambas.

## Vocabulario

`ShellExecRequest` (command, workdir?, timeoutMs?, stdoutMaxBytes?, signal?, stdin?, env?, dshEnv?, sandboxPolicy?) resuelve a `ShellExecSpec` (command, workdir, timeoutMs, stdoutMaxBytes, signal?, stdin?, env?, dshEnv?, sandboxPolicy) antes de la ejecución. `stdoutMaxBytes` es un presupuesto de captura de ejecución en primer plano de confianza para los consumidores que deben analizar stdout acotado completo; la herramienta bash orientada al modelo no lo expone. `sandboxPolicy` es opcional en la petición y obligatorio-pero-nulable en la spec resuelta: transporta el modo completo y la raíz del espacio de trabajo por llamada. La ruta de herramienta de sandbox lo resuelve desde la sesión llamante a través de `ctx.sandboxPolicy`; un llamante directo de ejecutor con sandbox recurre a la política de despliegue, mientras que un ejecutor sin sandbox transporta el campo y no confina nada.

El vocabulario de anulación de modo de sandbox por sesión (el evento `'sandbox/mode'`, el fold `effectiveSandboxMode(events)` y la ruta de escritura `setSandboxMode(session, mode)`) NO está aquí — es estado de política compartido por cada familia que aplica, propiedad de [`@deepseek-ai/dsh-sandbox-policy`](../../sandbox/sandbox-policy/). `run()` devuelve `ShellRunResult`; `start()` devuelve `ShellProcess`, cuyos métodos de lectura incremental y kill adapta `dsh-tool-bash` a un registro de tarea genérico. Un ejecutor con sandbox marca `ShellSandboxInfo` en los resultados en primer plano y en los manejadores de proceso asentados. Ver `src/types.ts` y [subsystems/shell.md](../../../docs/subsystems/shell.es.md).

`stdin` y el `env` ordinario los fijan los plugins intra-proceso (los bridges de hooks, los plugins nativos) para alimentar a un comando de hook con su payload JSON y los valores `CLAUDE_PROJECT_DIR`/`CLAUDE_PLUGIN_ROOT`. `dshEnv` es una superposición de confianza separada, restringida por tipo a claves gestionadas; el `DSH_ENV_PREFIX` exportado es la única fuente para ese namespace, su tipo plantilla `DshEnvironmentKey`, la limpieza del ejecutor, la validación del registro, los nombres integrados derivados y la guía del modelo. El bash del modelo usa la instantánea actual recogida por `ctx.shellEnv`. Las implementaciones eliminan las claves gestionadas heredadas y luego fusionan `dshEnv` después del `env` ordinario, de modo que un hecho actual omitido no pueda recurrir a estado ambiental obsoleto y una entrada de `env` no pueda desplazar un valor gestionado. La herramienta orientada al modelo no expone ninguno de estos como parámetro. Los tres permanecen opcionales en la spec resuelta; ausentes significan sin entrada/superposición. Ver [el Agent Note de bash-stdin-env](../../../.agents/notes/implemented/architecture/2026-06-30-bash-stdin-env-trusted-plugin-api.es.md) y [el Agent Note del entorno de sesión](../../../.agents/notes/implemented/feature/2026-07-10-agent-session-identity-and-log-location.es.md).

El `parseExitStatus` exportado (con `ParsedExitStatus`) es la mitad compartida del contrato de renderizado de las herramientas de shell: el inverso de los marcadores `[exit code: N]` / `[killed by signal: X]` que añaden `renderResult` de `dsh-tool-bash` y `renderPwshResult` de `dsh-tool-pwsh`. El `presentResult` de ambas herramientas lo usa para dividir el texto renderizado entre el cuerpo de salida de la tarjeta de terminal y su pill de estado de salida; vive con la Service Definition para que las dos herramientas nunca diverjan en el contrato de marcadores.

## Experiencia del modelo

Indirectamente, a través de `dsh-tool-bash`, que convierte la salida del ejecutor y los hechos de sandbox en guía y tokens retenidos de resultados de herramienta.

#### Efecto en la caché KV

Sin invalidación directa; el consumidor con nombre es dueño de cualquier cambio de prefijo de petición.

## Limitaciones conocidas y trabajo diferido

- **Sin vocabulario de entrada interactiva** — `stdin` se escribe una vez en el spawn y se cierra; el seam no tiene canal para alimentar una tarea en ejecución ni concepto de sesión PTY.
- **Los timeouts en primer plano son siempre del ejecutor** — un modo de plazo propiedad del llamante en el seam queda explícitamente diferido por [el Agent Note de política de timeout de llamadas de herramienta](../../../.agents/notes/implemented/architecture/2026-07-07-tool-call-timeout-policy.es.md).
