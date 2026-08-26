# @deepseek-ai/dsh-tool-pwsh

[English](README.md) | Español

La herramienta `pwsh` orientada al modelo se registra sobre el seam del ejecutor `ctx.shell`. Está pensada para composiciones de Windows en las que un ejecutor de PowerShell (p. ej. `@deepseek-ai/dsh-pwsh-local`) respalda `ctx.shell`; el contrato de la herramienta es del dialecto de PowerShell: rutas nativas `C:\...` y variables `$env:NAME`. Su comportamiento refleja llamada a llamada el de `dsh-tool-bash`: ejecución en primer plano y con `run_in_background` a través del runtime genérico de trabajos, el entorno gestionado `DSH_*` a través del registro compartido `shell-env`, la representación de la denegación del sandbox con la misma superficie de escalada `sandbox_permissions` en el mismo turno, y la historia de representación de marcadores/truncamiento de bash (una salida limpia no produce ningún marcador).

Requiere una implementación de ejecutor cargada y el plugin `shell-env`; la herramienta permanece pendiente hasta que ambos existan (`inject: ['tools', 'bash', 'systemPrompt', 'bashEnv']`).

La raíz del paquete expone únicamente el contrato del plugin de Cordis (`name`, `inject`, `Config`, `apply`); la representación de resultados (`src/render.ts`) y la adaptación de trabajos en segundo plano (`src/background.ts`) replican la estructura de la herramienta bash y siguen siendo accesibles a través de la exportación `./src/*` del paquete.

El plugin también aporta la sección de prompt `tool:pwsh` (orden 105): las salidas distintas de cero se notifican con marcadores `[exit code: N]`, y la interrupción en Windows se resuelve como salida 1 sin marcador de señal.

## Herramientas

### `pwsh`

| Argumento | Tipo | Notas |
|---|---|---|
| `command` | string (obligatorio) | Se ejecuta mediante `pwsh -Command`. No persiste estado entre llamadas: usa `workdir`, no `cd`. |
| `description` | string (obligatorio) | Resumen del comando de una línea en voz activa (5-10 palabras), solo para la visualización en la UI y los registros; no afecta a la ejecución. |
| `timeoutMs` | number | Anulación del tiempo de espera en milisegundos. El ejecutor aplica su valor predeterminado y su límite configurados. |
| `workdir` | string | Directorio de trabajo de esta llamada. Por defecto, el cwd de sesión del agent (agente) que llama (`session.header.cwd`), de modo que cada sesión se ejecuta en su propio espacio de trabajo; un `workdir` relativo se resuelve contra esa misma identidad. |
| `run_in_background` | boolean | Devuelve un id de trabajo inmediatamente; no se aplica tiempo de espera. |
| `sandbox_permissions` | string enum | Solo se anuncia cuando hay un ejecutor con sandbox montado (`ctx.shell.sandboxMode` definido). El modo de sandbox más amplio para un reintento único de un comando que el sandbox acaba de denegar — el modo más amplio más estrecho que baste, que exige `justification` y aprobación del usuario a través de `ctx.approval` ANTES de la ejecución. Una solicitud que no amplía o que no se puede aprobar falla en fail-closed sin ejecutar nada. |
| `justification` | string | Obligatoria con `sandbox_permissions`: una frase para el usuario que explique por qué este comando exacto necesita el acceso ampliado. |

`command`, `workdir` y `timeoutMs` se resuelven contra los valores predeterminados de configuración del ejecutor mediante `ctx.shell.resolve()` antes de la ejecución. El valor predeterminado de workdir se aplica en la capa de la herramienta a partir del `session.header.cwd` del agent que llama, ANTES de `resolve()` — el cwd por sesión debe proceder de `exec.agent`, ya que N sesiones comparten un único ejecutor; solo cuando no hay cwd de sesión disponible recurre el ejecutor a su propia configuración o a `process.cwd()`.

### Entorno de shell gestionado

Cada llamada de pwsh al modelo, en primer plano o en segundo plano, recibe un entorno `DSH_*` de confianza recién recopilado a través del registro compartido [`dsh-shell-env`](../shell-env/): `DSH_HOME` (el home absoluto de Harness), `DSH_SHELL=1`, el `DSH_SESSION_ID` del agent y `DSH_SESSION_JSONL` cuando el backend de persistencia activo localiza uno. Los plugins que aportan hechos `DSH_*` a `ctx.shellEnv` se aplican a las llamadas de pwsh exactamente igual que a las de bash. La instantánea pasa por el canal dedicado `ShellExecRequest.dshEnv`; `process.env` nunca se modifica. La descripción enseña la convención genérica `$env:DSH_*` en lugar de nombrar variables específicas de la persistencia.

El texto del resultado contiene stdout, una sección opcional `[stderr]` y, a continuación, los marcadores aplicables de truncamiento, de denegación del sandbox (con la pista de escalada del mismo turno cuando la composición anuncia escalada), de tiempo de espera, de señal y de salida. Una salida limpia (0, sin señal) no produce marcador; un cuerpo vacío se representa como `(no output)`. El truncamiento enlaza un archivo de spill completo y seguro o informa de que no está disponible. El tiempo de espera se informa independientemente del estado final de salida; la salida distinta de cero sigue siendo un resultado interpretado por el modelo, no un `isError`. Windows informa de la terminación forzada como salida 1 sin señal, por lo que `[killed by signal: …]` solo existe en POSIX. Solo los fallos de infraestructura — errores de spawn y abortos (`tool call aborted`) — producen `isError`.

El éxito canónico es `{ kind: 'foreground', ...ShellRunResult }` para un proceso en primer plano completado (con los hechos de `sandbox` del ejecutor — `mode`/`denied`, y `enforcement`/`runnerFailed` opcionales — proyectados cuando están presentes) o `{ kind: 'background', jobId }` para una tarea publicada. El renderizador conserva exactamente `started background job <id>` para los acuses de recibo en segundo plano; los consumidores programáticos usan los campos tipados sin analizar el texto representado.

Cuando `run_in_background` es true, este plugin precomprueba `ctx.jobs.start()` antes del spawn, registra al agent llamante como propietario y adapta el manejador `ShellProcess` devuelto a ganchos genéricos de cancelación/finalización/salida incremental. El runtime de trabajos es dueño de los ids, del aislamiento entre sesiones, de los avisos de finalización, de la espera y de la limpieza al desechar; este plugin solo asigna los hechos de salida de pwsh al resultado y al detalle de resultado del trabajo. `enableRunInBackground: false` elimina el parámetro y rechaza en tiempo de ejecución una llamada forzada en segundo plano.

## Presentación en la UI

La herramienta es dueña de su intención de renderizado `presentCall`/`presentResult`. Una llamada en primer plano es una tarjeta `terminal` con el comando, la descripción y el cwd opcional; una llamada con `run_in_background` es una tarjeta `generic` con el comando crudo, igual que la presentación en segundo plano de la herramienta bash. Un resultado en primer plano completado también es una tarjeta `terminal`: el marcador de salida se convierte en la píldora de estado de salida de la tarjeta (`exitCode`/`signal`), y el cuerpo sin marcador es la salida de la tarjeta — exactamente la historia de tarjeta terminal de la herramienta bash, a través del análisis compartido de estado de salida de `@deepseek-ai/dsh-shell`. Los acuses de recibo en segundo plano y los errores de ejecución siguen siendo tarjetas `generic` con la salida representada en un recuadro `console`. Estos presentadores son puros y seguros frente a la reproducción.

## Experiencia del modelo

### System prompt

#### Lo que ve el modelo

Cada solicitud dentro del ámbito de registro de este plugin contiene la guía de pwsh siguiente. Las restricciones de herramientas con ámbito pueden ocultar el schema sin eliminar esta sección registrada de forma independiente.

##### Guía de Pwsh

```markdown
Non-zero exits are reported as `[exit code: N]` markers; investigate failures before moving on. On Windows a killed process settles as `[exit code: 1]` without a signal marker; treat a bare exit 1 after an interruption as a termination, not a command failure.
```

#### Efecto en tokens

Coste fijo pequeño de entrada por solicitud mientras el plugin está activo.

#### Efecto en la caché KV

Estable en el prefijo mientras el ámbito de registro y el texto del prompt no cambien. La activación o el desecho del plugin pueden invalidar la reutilización de esta sección de prompt.

### Schemas de herramientas

#### Lo que ve el modelo

El modelo ve el [schema `pwsh`](../../../docs/tool-catalog.es.md#deepseek-aidsh-tool-pwsh) generado. Las restricciones de herramientas con ámbito de agent pueden eliminar la definición para ese agent.

#### Efecto en tokens

Coste fijo de schema en cada solicitud en la que la herramienta es visible.

#### Efecto en la caché KV

Estable en el prefijo mientras la visibilidad y la definición de la herramienta no cambien. Una restricción o un cambio de configuración pueden invalidar la reutilización desde el primer token modificado.

### Resultado en primer plano

#### Lo que ve el modelo

El renderizador emite la cola de stdout dependiente de los datos y, a continuación, `[stderr]` opcional y la cola de stderr. Las líneas condicionales son exactamente `[output truncated; full output: <path>]`, `[sandbox: file access denied under <mode> mode]` más la pista de escalada `[sandbox: escalation available — …]` (solo cuando la composición anuncia escalada), `[timed out after <timeoutMs>ms]`, `[killed by signal: <signal>]` y `[exit code: <exitCode>]` (solo salidas distintas de cero); un cuerpo vacío se representa como `(no output)`.

#### Efecto en tokens

Cero tokens de resultado antes de una llamada. La salida está acotada por flujo, mientras que cada línea emitida permanece en el historial hasta la compactación.

#### Efecto en la caché KV

Solo añadido; el contenido recién visible sigue el prefijo de solicitud reutilizable y no invalida las entradas existentes de la caché KV.

### Resultado en segundo plano

#### Lo que ve el modelo

Un arranque en segundo plano se representa exactamente como `started background job <id>`; las lecturas y el estado posteriores fluyen a través de las herramientas genéricas `job_output`/`job_kill`, incluido el aviso de spill de lectura con pérdida cuando el truncamiento en memoria descartó bytes no leídos.

#### Efecto en tokens

El acuse de recibo es una línea corta fija; la salida del trabajo está acotada por lectura.

#### Efecto en la caché KV

Solo añadido; el contenido recién visible sigue el prefijo de solicitud reutilizable y no invalida las entradas existentes de la caché KV.

### Errores de la herramienta

#### Lo que ve el modelo

Los fallos de validación e infraestructura se normalizan como `Error: <message>`. Los mensajes estables de este paquete son `invalid command: expected a non-empty string`, `invalid description: expected a non-empty string`, `invalid timeoutMs: expected a positive number, got <value>`, `invalid escalation: sandbox_permissions requires a justification`, `invalid escalation: justification is only valid together with sandbox_permissions`, `invalid justification: expected a non-empty sentence`, `sandbox_permissions is not available in this composition (no sandboxing executor to escalate)`, los fallos de escalada compartidos (no estrictamente más amplio / sin servicio de aprobación / sin agent al que enrutar / sin canal de aprobación / rechazado por el usuario / cancelado), `run_in_background is disabled for this deployment (enableRunInBackground: false)`, `background jobs unavailable: load @deepseek-ai/dsh-jobs and @deepseek-ai/dsh-tool-jobs` y `tool call aborted`.

#### Efecto en tokens

Solo la llamada fallida añade estos tokens retenidos; una llamada abortada no añade salida de comando.

#### Efecto en la caché KV

Solo añadido; el contenido recién visible sigue el prefijo de solicitud reutilizable y no invalida las entradas existentes de la caché KV.

## Limitaciones conocidas y trabajo diferido

- **Modo de lenguaje y captura por named pipe bajo el sandbox de Windows** — bajo el [sandbox ACL de Windows](../../sandbox/sandbox-windows-acl/README.es.md), un pwsh de solo lectura arranca en ConstrainedLanguage porque la denegación de escritura en su directorio temporal hace que la sonda de AppLocker de PowerShell falle en fail-closed: `Add-Type`, los estáticos .NET no básicos (`[System.IO.*]::`, `[math]::`), los objetos COM y la reflexión fallan con errores de «only core types», y el modo no se puede elevar desde dentro. El directorio temporal privado de workspace-write permite que la sonda se complete, por lo que permanece en FullLanguage salvo que la política del host diga lo contrario. Ambos modos confinados deniegan las aperturas de named pipe, por lo que un spawn de stdio por tubería dentro de un comando confinado falla con EPERM. La descripción de la herramienta enseña ambos contratos al modelo; el README del backend es dueño de las limitaciones completas.
- **Sin shell persistente** — cada llamada arranca un `pwsh -Command` nuevo; la contrapartida de shell persistente es [`@deepseek-ai/dsh-tool-pwsh-persistent`](../tool-pwsh-persistent/README.es.md), que mantiene vivo un pwsh con ámbito de propietario entre llamadas en hosts de Windows (ConPTY) y POSIX con pwsh.
- **Contrato del dialecto de PowerShell** — el modelo debe escribir PowerShell (rutas nativas, variables `$env:`), no bash; no hay traducción de dialecto.
- **La identidad del cwd de sesión no está canonizada** — la base de workdir es el cwd del encabezado de sesión tal cual, a diferencia de la identidad canonizada por la raíz del sandbox de la herramienta bash. Bajo un ejecutor confinante, la raíz del espacio de trabajo de la política SÍ está canonizada (por el servicio de política compartido), por lo que workdir y la raíz de confinamiento pueden divergir cuando el cwd crudo de la sesión difiere de su forma canónica — una brecha de paridad diferida a la extracción de la base compartida de herramientas de shell.
