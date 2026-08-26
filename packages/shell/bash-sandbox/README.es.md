# @deepseek-ai/dsh-bash-sandbox

[English](README.md) | Español

Service Provider que consume sandbox para el seam de ejecutor [`@deepseek-ai/dsh-shell`](../shell/). Cárgalo **en lugar de** `@deepseek-ai/dsh-bash-local`, junto con un provider de [`ctx.sandbox`](../../sandbox/sandbox/) (p. ej. [`@deepseek-ai/dsh-sandbox-local`](../../sandbox/sandbox-local/)) y un [`ctx.sandboxPolicy`](../../sandbox/sandbox-policy/) (que es dueño del modo predeterminado + la raíz del workspace, compartidos con el sistema de archivos con sandbox) — no se necesita ningún plugin de herramienta alternativo; `dsh-tool-bash` detecta la capacidad `sandboxMode` del ejecutor y añade los campos de escalada.

La raíz del paquete exporta el plugin `SandboxBashExecutor` por defecto y con nombre, además de su `Config`; los helpers de clasificación de resultados permanecen internos.

Cada comando se confina entregando al provider el argv exacto `['bash', '-c', command]` que este ejecutor está a punto de lanzar y lanzando directamente el argv devuelto. Con los runners nativos incluidos, el Bash interno conserva la semántica de shell y evalúa `BASH_ENV` solo después de que el runner establece el confinamiento. QUÉ runner de plataforma lo confina — y si hay alguno utilizable (fail closed con un error estructurado `SANDBOX_UNAVAILABLE`, nunca una ejecución no confinada silenciosa) — es cosa del provider; este paquete es dueño solo del lado bash.

| Modo | Efectos sobre archivos |
|---|---|
| `read-only` (predeterminado) | Ninguna escritura en ningún sitio (de `/dev`, solo el nodo `/dev/null` es escribible, así que `>/dev/null` sigue funcionando) |
| `workspace-write` | Escrituras solo bajo `workspaceRoot` + `/tmp` (efímero bajo bwrap, el `/tmp` del host bajo Landlock, `/private/tmp` más el dir temporal por usuario bajo Seatbelt) |
| `danger-full-access` | Sin confinamiento; el provider nunca se consulta. Los resultados en primer plano llevan `sandbox: { mode, denied: false }`; los manejadores de procesos en segundo plano no llevan hechos de sandbox. |

Semántica:

- **Las denegaciones son hechos de resultado.** Una ejecución fallida cuyo stderr lleva el dialecto de denegación del propio backend seleccionado — las firmas que el provider estampa en cada wrap (texto EROFS bajo bwrap, EACCES bajo Landlock, EPERM bajo Seatbelt) — se informa como `ShellRunResult.sandbox.denied: true` (clasificación conservadora, leída de la cola de stderr recogida); toda ejecución CONFINADA lleva además el modo bajo el que se ejecutó (`result.sandbox.mode`) y la completitud del enforcement del provider (`result.sandbox.enforcement`: `full`, o `partial` con un ABI Landlock antiguo).
- **La ruta del runner o el syscall deben coincidir.** Antes de que un proceso arranque, un rechazo se atribuye al runner solo cuando el workdir propiedad del llamador es utilizable de forma independiente y Node informa `ENOENT` o `EACCES` con un `error.path` igual a provider argv[0] o, cuando `error.path` está ausente, un `syscall: 'spawn <runner>'` exacto. Una ruta presente exige además `syscall: 'spawn'` o el `spawn <runner>` exacto. Esto cubre un runner ausente, un runner no ejecutable o un script ejecutable cuyo intérprete shebang no está disponible. Un `syscall: 'spawn'` pelado sin ruta de error exacta, cualquier otro código, un workdir inválido o inutilizable, un fallo de recursos, un syscall no relacionado o un rechazo no estructurado conservan la semántica de fallo de arranque de comando del ejecutor local. La ejecución en primer plano lanza `SANDBOX_UNAVAILABLE` con el detalle de spawn original, mientras que la resolución asíncrona en segundo plano estampa `runnerFailed: true` y `denied: false`. Si un `SubprocessRuntime` lanza de forma síncrona la misma forma `ENOENT`/`EACCES` identificadora del runner, el arranque en segundo plano lanza `SANDBOX_UNAVAILABLE`; otros errores síncronos se propagan sin cambios. Después de que un proceso arranca, la comprobación opcional de código de salida de una regla y una línea fatal de stderr restante deben coincidir ambas tras las exclusiones exactas de líneas informativas. Una coincidencia tiene prioridad sobre la denegación; la ejecución en primer plano lanza `SANDBOX_UNAVAILABLE` con la línea fatal coincidente, mientras que un proceso en segundo plano ya resuelto estampa `process.sandbox.runnerFailed`, que el producer de bash renderiza a través del `job_output` genérico. Los manejadores en segundo plano confinados conservan sus hechos de modo/enforcement y liberan la contabilidad por proceso en cualquiera de las dos vías.
- **Fallback de despliegue, política por llamada.** [`ctx.sandboxPolicy`](../../sandbox/sandbox-policy/) resuelve una `SandboxExecutionPolicy` completa para cada llamada de herramienta: la sesión llamante aporta su override de modo y su raíz de cwd inmutable, mientras que la configuración de despliegue aporta los fallbacks para las llamadas sin agent. Una escalada aprobada cambia solo el modo de esa política; su raíz de sesión permanece adjunta. `resolve()` transporta la política a la spec, de modo que los comandos solapados de distintos proyectos se ejecutan, clasifican e informan bajo sus propias raíces y modos. El hecho de capacidad `ctx.shell.sandboxMode` informa del predeterminado configurado para que la capa de herramientas anuncie la escalada solo cuando este ejecutor está montado; la descripción estática de la herramienta bash es dueña por separado de la guía de denegación y escalada.
- **Solo efectos sobre archivos.** El vocabulario de modos reclama únicamente efectos sobre archivos. La red permanece sin restricciones; la visibilidad de procesos es específica del backend y la documenta [`dsh-sandbox-local`](../../sandbox/sandbox-local/).
- La mecánica de procesos (spawn, kills de grupos de procesos, recogida/spill de salida, manejadores en segundo plano, limpieza de credenciales) se hereda de [`dsh-bash-local`](../bash-local/); la selección del runner vive en [`dsh-sandbox-local`](../../sandbox/sandbox-local/).

Solo denegación en el seam: una denegación es un hecho informado, y este ejecutor nunca negocia permisos por sí mismo — la pregunta de aprobación vive en la capa de herramientas (`dsh-tool-bash`), que impulsa el override que este paquete honra.

```yaml
- id: sandbox
  name: '@deepseek-ai/dsh-sandbox-local'
- id: sandbox-policy
  name: '@deepseek-ai/dsh-sandbox-policy'
  config:
    mode: read-only
    workspaceRoot: !!js process.cwd() # fallback for calls without a session cwd
- id: bash
  name: '@deepseek-ai/dsh-bash-sandbox'
```

## Model Experience

### Schema de la herramienta bash, de forma indirecta

#### Lo que ve el modelo

Los [schemas generados de `dsh-tool-bash`](../../../docs/tool-catalog.es.md#deepseek-aidsh-tool-bash) son la línea base. Al anunciar un `sandboxMode` que confina, este backend aumenta `bash` con `sandbox_permissions` usando el enum `workspace-write` | `danger-full-access` y con `justification`. El propietario de la política contribuye por separado el contexto actual `sandbox:policy` neutro respecto a capacidades.

#### Efecto en tokens

Incremento de schema fijo y pequeño en las peticiones donde `bash` es visible, más la cláusula de política actual propiedad de `dsh-sandbox-policy`.

#### Efecto de KV Cache

Un cambio de política vigente anexa una instantánea de contexto completa renderizada por el propietario después del historial retenido, preservando byte a byte el prefijo system/history existente. Cambiar las capacidades del ejecutor altera el schema de `bash`.

### Resultado de la herramienta bash, de forma indirecta

#### Lo que ve el modelo

Tras la salida acotada ordinaria, una llamada denegada anexa exactamente `[sandbox: file access denied under <mode> mode]`. Cuando la escalada está disponible, a continuación anexa `[sandbox: escalation available — retry this exact command once with sandbox_permissions (the narrowest wider mode that suffices) + justification; the approval prompt asks the user]`. Un fallo de runner en segundo plano ya resuelto anexa en su lugar `[sandbox: the sandbox runner itself failed under <mode> mode — the command did not run; this is a sandbox problem, not a command failure]`.

#### Efecto en tokens

Cero tokens adicionales en una ejecución permitida sin incidentes más allá de la salida ordinaria. La denegación o el fallo añade el marcador condicional entre comillas, retenido hasta la compactación.

#### Efecto de KV Cache

Solo anexión; el contenido recién visible sigue al prefijo de petición reutilizable y no invalida las entradas existentes de la KV cache.

### Error de la herramienta bash, de forma indirecta

#### Lo que ve el modelo

Si ningún runner puede hacer cumplir un modo confinado, la llamada en primer plano propaga el [error `SANDBOX_UNAVAILABLE` propiedad de `dsh-sandbox`](../../sandbox/sandbox/README.es.md#confinement-error-indirectly). Un fallo de spawn atribuible al runner aporta el error de spawn original como detalle; un rechazo sin evidencia de ruta `ENOENT`/`EACCES` ni de syscall que nombre argv[0] sigue siendo un error ordinario de arranque de comando. Un fallo de runner ya resuelto aporta la línea fatal de stderr coincidente y preserva la recogida original de stderr. Cuando está presente, el `Runner failure: <detail>` anexado es el diagnóstico autoritativo; el texto anterior de instalación del backend es el prefijo genérico de `SANDBOX_UNAVAILABLE`.

#### Efecto en tokens

El texto de error condicional es visible para esa llamada y se retiene en el historial hasta la compactación.

#### Efecto de KV Cache

Solo anexión; el contenido recién visible sigue al prefijo de petición reutilizable y no invalida las entradas existentes de la KV cache.

## Limitaciones conocidas y trabajo diferido

- **El confinamiento cubre solo efectos sobre archivos** — la restricción de red y una garantía uniforme de visibilidad de procesos están ausentes, así que los modos no son un sandbox de seguridad de propósito general.
- **Las denegaciones se infieren del stderr del comando fallido** — las firmas de los backends hacen portable la inferencia, pero un error de aplicación coincidente puede clasificarse como denegación y una denegación omitida de la cola retenida puede pasar desapercibida.
- **Un fallo de runner en segundo plano observado de forma asíncrona no tiene canal de error inmediato** — se registra en el proceso ya resuelto y sale a la superficie cuando el llamador lee la tarea genérica con `job_output`; en cambio, un throw síncrono de `SubprocessRuntime` que nombre la ruta del runner hace fallar `start()` de inmediato.
- **`danger-full-access` elude deliberadamente `ctx.sandbox`** — es un modo no confinado explícito, no un perfil de sandbox más amplio.
