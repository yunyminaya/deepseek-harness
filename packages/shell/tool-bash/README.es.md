# @deepseek-ai/dsh-tool-bash

[English](README.md) | Español

La herramienta `bash` orientada al modelo registrada sobre el seam de ejecución `ctx.shell`. La ejecución en primer plano permanece detrás de ese seam; un manejador de proceso en segundo plano se registra en el runtime genérico `ctx.jobs` y se controla a través de `job_output`, `job_list` y `job_kill` de `@deepseek-ai/dsh-tool-jobs`.

Requiere un Service Provider de ejecutor cargado (p. ej. `@deepseek-ai/dsh-bash-local`) y el registro [`@deepseek-ai/dsh-shell-env`](../shell-env/README.es.md); el plugin permanece pendiente hasta que existe cada servicio inyectado (`inject: ['tools', 'bash', 'systemPrompt', 'bashEnv']`). El contrato de herramienta es del dialecto bash — monta un ejecutor que analice bash.

La raíz del paquete expone solo el contrato de plugin de Cordis (`name`, `inject`, `Config`, `apply`); el renderizado de resultados y la adaptación de procesos en segundo plano permanecen internos al paquete.

El plugin también contribuye la sección de prompt `tool:bash` (orden 105): comprueba el marcador `[exit code: N]` en cada resultado e investiga los fallos antes de continuar.

## Herramientas

### `bash`

| Argumento | Tipo | Notas |
|---|---|---|
| `command` | string (obligatorio) | Se ejecuta vía `bash -c`. No persiste ningún estado entre llamadas — usa `workdir`, no `cd`. |
| `description` | string (obligatorio) | Resumen de una línea en voz activa del comando (5-10 palabras), solo para la visualización en UI/logs — sin efecto sobre la ejecución. |
| `timeoutMs` | number | Anulación de timeout en milisegundos. El ejecutor aplica su valor por defecto y su tope configurados. |
| `workdir` | string | Directorio de trabajo para esta llamada. Por defecto, la identidad de sistema de archivos del cwd de sesión del agent llamante (`session.header.cwd`), de modo que cada sesión se ejecuta en su propio espacio de trabajo; un `workdir` relativo se resuelve contra esa misma identidad. |
| `run_in_background` | boolean | Devuelve un id de job inmediatamente; no se aplica ningún timeout. |
| `sandbox_permissions` | string enum | SE ANUNCIA SOLO cuando el ejecutor montado aplica sandbox (`ctx.shell.sandboxMode` informa un valor por defecto confinante): el modo más amplio que necesita un comando denegado, del vocabulario objetivo cerrado `workspace-write`/`danger-full-access` (nunca se reduce al valor por defecto del ejecutor — el modo efectivo es por sesión; el ensanchamiento estricto se comprueba en la ejecución contra él, y una petición que no ensancha falla sin preguntar a nadie). |
| `justification` | string | Obligatoria junto con `sandbox_permissions` (cada una sin la otra es un error de validación): una frase para el usuario explicando por qué este comando exacto necesita el acceso más amplio. |

`command`, `workdir` y `timeoutMs` se resuelven contra los valores por defecto de configuración del ejecutor vía `ctx.shell.resolve()` antes de la ejecución, de modo que la Service Definition (`ShellExecSpec`) recibe valores `workdir`/`timeoutMs` explícitos. El valor por defecto de workdir se aplica en la capa de herramientas a partir del `session.header.cwd` del agent llamante ANTES de `resolve()` — el cwd por sesión debe venir de `exec.agent`, ya que N sesiones comparten un ejecutor; solo cuando no hay cwd de sesión disponible recurre el ejecutor a su propia configuración / `process.cwd()`. Cuando hay política de sandbox, la herramienta reutiliza su ya canónico `workspaceRoot` como base de workdir para que el confinamiento y el lanzamiento del proceso no puedan resolver la misma grafía de sesión de forma distinta.

### Entorno de shell gestionado

Cada llamada bash del modelo, en primer plano o en segundo plano, recibe un entorno `DSH_*` de confianza recién recogido a través del registro compartido [`dsh-shell-env`](../shell-env/README.es.md): `DSH_HOME` (el home absoluto del Harness), `DSH_SHELL=1`, el `DSH_SESSION_ID` del agent y `DSH_SESSION_JSONL` cuando el backend de persistencia activo localiza uno. El contrato del registro — registro de contributors, fallo en voz alta por clave duplicada/no declarada, las reservas integradas y el ejemplo de contributor — vive en el README de ese paquete. La instantánea pasa por el canal dedicado `ShellExecRequest.dshEnv`; el ejecutor local elimina todo `DSH_*` heredado antes de fusionarla, de modo que los harnesses anidados y los agents padre/hijo concurrentes no puedan filtrar identidades obsoletas, y `process.env` nunca se modifica. La descripción de la herramienta enseña la convención genérica `$DSH_*` en lugar de nombrar variables específicas de persistencia o añadir una sección permanente de system prompt.

El texto del resultado contiene stdout, una sección opcional `[stderr]` y después los marcadores aplicables de denegación de sandbox, timeout, señal, código de salida y truncación. El timeout se informa independientemente del estado de salida final; la salida distinta de cero sigue siendo un resultado interpretado por el modelo, no un `isError`. La truncación enlaza un archivo spill completo seguro o informa de que no está disponible. Solo los fallos de infraestructura como los errores de spawn y los abortos producen `isError`.

El éxito canónico es `{ kind: 'foreground', ...ShellRunResult }` para un proceso en primer plano completado o `{ kind: 'background', jobId }` para una tarea publicada. El renderizador Native preserva el texto anterior, incluido exactamente `started background job <id>`; los consumidores programáticos usan los campos tipados sin analizar esas cadenas. Los topes de flujo del ejecutor siguen siendo límites de adquisición en `ShellRunResult` y transportan sus rutas de spill.

Cuando `run_in_background` es true, este plugin pre-comprueba `ctx.jobs.start()` antes de lanzar el spawn, registra al agent llamante como propietario y adapta el manejador `ShellProcess` devuelto en hooks genéricos de cancelación/cierre/salida incremental. El runtime de jobs es dueño de los ids, el aislamiento entre sesiones, los avisos de finalización, la espera y la limpieza del disposal; este plugin solo mapea los hechos de salida/sandbox de bash en la salida del job y el detalle del resultado. `enableRunInBackground: false` elimina el parámetro y rechaza una llamada forzada en segundo plano en el momento de la ejecución.

## Presentación en la UI

La herramienta es dueña de su intención de renderizado `presentCall`/`presentResult`. Una llamada en primer plano es una tarjeta de terminal con comando, descripción, cwd, salida y estado de salida analizado. Como la tarjeta muestra la salida como su propia pill, el marcador `[exit code: N]` / `[killed by signal: …]` que consume el análisis sale de la salida; todos los demás marcadores (truncación, timeout, sandbox) permanecen en ella. Un inicio en segundo plano es una tarjeta de ejecución genérica porque devuelve solo un id de job; las herramientas genéricas `job_*` son dueñas de sus propias tarjetas. Estos presentadores son puros y seguros ante replay.

## La herramienta construye su petición solo con argumentos con nombre

`ShellExecRequest` transporta `stdoutMaxBytes`, `stdin`, el `env` ordinario y el `dshEnv` gestionado, opcionales, usados por los plugins intra-proceso de confianza y por el registro de entorno de esta herramienta. La herramienta orientada al modelo no expone ninguno de `stdoutMaxBytes`, `stdin` o `env`: construye peticiones a partir de los campos con nombre command/workdir/timeout/signal/sandbox más el `dshEnv` recogido por el registro. Las claves extra del modelo se ignoran y no pueden reemplazar valores gestionados. La sintaxis de shell aporta el comportamiento equivalente a nivel de comando, mientras que el ejecutor local limpia las credenciales ambientales y los valores `DSH_*` obsoletos. Ver el [Agent Note de stdin/env](../../../.agents/notes/implemented/architecture/2026-06-30-bash-stdin-env-trusted-plugin-api.es.md).

## Permisos y escalada

Los comandos se ejecutan con toda la autoridad del ejecutor a menos que un ejecutor con sandbox ([`dsh-bash-sandbox`](../bash-sandbox/)) los confine — el sandbox de solo denegación informa las denegaciones como hechos de resultado, renderizados aquí como marcador de denegación; la política allow/deny/ask por llamada es el waterfall de `tools/pre-execute` (ver docs/architecture.md).

Las llamadas bash con escalada resuelven `ctx.approval` antes de la ejecución. `allowed-once` aplica el modo solicitado solo a esa llamada; el rechazo, la cancelación, la no disponibilidad o la ausencia de contexto de aprobación no ejecutan nada y devuelven un error distinto. Ante una denegación real, el modelo puede reintentar el mismo comando una vez en el mismo turno con el modo y la justificación más estrechos suficientes; el propio prompt de aprobación es el paso de consentimiento. La escalada nunca es especulativa, y una aprobación deshabilitada o rechazada es definitiva. El [Agent Note de sandbox](../../../.agents/notes/implemented/feature/2026-07-06-sandbox.es.md) es dueño de la justificación.

## Cambio de modo por sesión

Para los ejecutores con sandbox, cada llamada resuelve el modo como escalada de un solo uso, luego anulación de sesión y luego valor por defecto del ejecutor. Las llamadas sin sandbox y sin agent no transportan anulación de sesión. El propietario de la política contribuye el modo permanente actual neutral a la capacidad; los resultados de denegación siguen siendo dueños del modo efectivo específico de la operación y de la guía de reintento. Ver el [fold `dsh-shell`](../shell/README.es.md) y el [contrato de cambio de sandbox](../../../.agents/notes/implemented/feature/2026-07-06-sandbox.es.md).

## Experiencia del modelo

### System prompt

#### Lo que ve el modelo

Cada petición en el ámbito de registro de este plugin contiene la guía bash siguiente. El propietario de la política contribuye el estado actual de sandbox a través de su contexto de runtime seguro para caché en lugar de cambiar esta sección. Las restricciones de herramienta con ámbito pueden ocultar los schemas sin eliminar esta sección registrada de forma independiente.

##### Guía bash

```markdown
Check the [exit code: N] marker on every bash result; investigate failures before moving on.
```

#### Efecto en tokens

Costo de entrada fijo pequeño por petición mientras el plugin está activo, sin cambios por el modo de sandbox ni los cambios de modo.

#### Efecto en la caché KV

Estable por prefijo mientras el ámbito de registro y el texto de prompt permanecen sin cambios. La activación o el disposal del plugin pueden invalidar la reutilización de esta sección de prompt; los cambios de modo de sandbox no.

### Schemas de herramienta

#### Lo que ve el modelo

El modelo ve el [schema `bash`](../../../docs/tool-catalog.es.md#deepseek-aidsh-tool-bash) generado. `run_in_background` aparece solo cuando este productor lo habilita; `sandbox_permissions` y `justification` aparecen solo cuando el ejecutor montado anuncia sandbox. Las restricciones de herramienta con ámbito de agent pueden eliminar la definición para ese agent.

#### Efecto en tokens

Costo de schema fijo en cada petición donde las herramientas son visibles; el soporte de sandbox añade los campos de escalada y su párrafo de descripción condicional.

#### Efecto en la caché KV

Estable por prefijo mientras la visibilidad, el soporte de segundo plano y las capacidades de sandbox del ejecutor permanecen sin cambios. Una restricción, un cambio de configuración o un cambio de ejecutor pueden invalidar la reutilización desde la primera definición de herramienta cambiada.

### Resultado en primer plano

#### Lo que ve el modelo

El renderizador emite la cola de stdout dependiente de los datos, luego el `[stderr]` opcional y la cola de stderr. Sin salida emite exactamente `(no output)`. Las líneas condicionales son exactamente `[output truncated; full output: <path-or-(unavailable)>]`, `[sandbox: file access denied under <mode> mode]`, `[timed out after <timeoutMs>ms]`, `[killed by signal: <signal>]` y `[exit code: <exitCode>]`; las líneas de escalada de sandbox y de fallo del runner se citan en [`dsh-bash-sandbox`](../bash-sandbox/README.es.md).

#### Efecto en tokens

Cero tokens de resultado antes de una llamada. La salida está acotada por flujo, mientras cada línea emitida permanece en el historial hasta la compactación.

#### Efecto en la caché KV

De solo añadido; el contenido recién visible sigue el prefijo de petición reutilizable y no invalida las entradas existentes de la caché KV.

### Contexto y resultados de jobs en segundo plano

#### Lo que ve el modelo

El inicio devuelve exactamente `started background job <jobId>`. Este productor suministra al runtime de jobs genérico salida incremental de proceso, el opcional `[some output was dropped from memory; full output: <paths-or-(unavailable)>]`, hechos de sandbox y detalle terminal como `exit code: <exitCode>` o `signal: <signal>`. [`dsh-tool-jobs`](../../jobs/tool-jobs/README.es.md) es dueño de la línea de estado visible, el aviso de finalización, el listado y la respuesta de cancelación.

#### Efecto en tokens

El acuse de inicio es pequeño y se retiene; la salida recogida es dependiente de los datos y está acotada por los búferes de flujo del ejecutor. Las lecturas consumidoras no repiten la salida previa.

#### Efecto en la caché KV

De solo añadido; el contenido recién visible sigue el prefijo de petición reutilizable y no invalida las entradas existentes de la caché KV.

### Errores de herramienta

#### Lo que ve el modelo

Los fallos de validación y política se normalizan como `Error: <message>`. Los mensajes estables de este paquete son `invalid command: expected a non-empty string`, `invalid description: expected a non-empty string`, `invalid timeoutMs: expected a positive number, got <value>`, `invalid escalation: sandbox_permissions requires a justification`, `invalid escalation: justification is only valid together with sandbox_permissions`, `invalid justification: expected a non-empty sentence`, `background execution is disabled for this bash tool`, `background jobs unavailable: load @deepseek-ai/dsh-jobs and @deepseek-ai/dsh-tool-jobs`, `sandbox_permissions is not available in this composition (no sandboxing executor to escalate)`, `sandbox escalation to "<mode>" is not strictly wider than this call's current "<mode>" mode`, las variantes de disponibilidad/rechazo/cancelación de aprobación y `tool call aborted`.

#### Efecto en tokens

Solo la llamada que falla añade estos tokens retenidos; una escalada rechazada no añade salida de comando porque el comando no se ejecuta.

#### Efecto en la caché KV

De solo añadido; el contenido recién visible sigue el prefijo de petición reutilizable y no invalida las entradas existentes de la caché KV.

## Limitaciones conocidas y trabajo diferido

- **Las pills de salida en replay se analizan del texto del resultado** — una salida cuya última línea es exactamente `[exit code: N]` / `[killed by signal: …]` muestra una pill incorrecta en el replay de sesión y pierde esa línea del cuerpo de la tarjeta, porque el análisis la trata como el marcador que consume; residual conocido solo de visualización.
- **La herramienta `bash` opta por no usar los presupuestos de `timeout-policy`** — mantiene la ruta `BASH_TIMEOUT` propiedad del ejecutor, según [el Agent Note de política de timeout de llamadas de herramienta](../../../.agents/notes/implemented/architecture/2026-07-07-tool-call-timeout-policy.es.md).
- **Los procesos en segundo plano no tienen timeout de ejecutor** — los llamantes deben usar `job_kill`, o confiar en el disposal del propietario/servicio, cuando el trabajo ya no importa.
