# @deepseek-ai/dsh-sandbox

[English](README.md) | Español

Service Definition del sandbox de procesos. Es el dueño del contrato de servicio `ctx.sandbox` ([`SandboxProvider`](src/index.ts)) y del vocabulario de confinamiento que comparte el harness: `SandboxMode` (`read-only` / `workspace-write` / `danger-full-access`, solo efectos de archivo), `SandboxEnforcement` (`full` / `partial`, según el ABI del kernel), `SandboxExecutionPolicy` (el modo por llamada completo + la raíz del workspace), `SandboxPolicy` (su subconjunto confinado) y el error de fallo cerrado `SANDBOX_UNAVAILABLE`. Como rol de Service Definition de la [división de capability seams](../../../.agents/notes/implemented/architecture/2026-06-13-capability-seams.md), depende solo de cordis (+ la base de errores del harness), nunca de un backend.

El contrato en una línea: `ctx.sandbox.confine(argv, policy)` devuelve el argv que hay que lanzar EN LUGAR del tuyo — envuelto para que el proceso (y todo lo que lance) corra confinado — más la completitud de cumplimiento del backend seleccionado, el dialecto de denegación (`denialSignatures`) y la evidencia estructurada de fallo del runner (`runnerFailureRules`); cuando ningún backend es utilizable, lanza una excepción en lugar de pasar el argv sin confinar. El [catálogo de tipos del núcleo](../../../docs/subsystems/sandbox.es.md#wrapped-argv-and-classification-dialects) es el dueño de la forma exacta del clasificador.

La política viaja en la llamada, no en el provider: dos consumidores pueden confinar con políticas distintas en el mismo instante (bash en `read-only` mientras un child agent confinado mantiene escribible su directorio de estado), y un reintento escalado aprobado es solo una llamada nueva con una política más amplia.

**Confinamiento solo en el mismo mundo.** Un backend comparte el filesystem y el kernel del host (`bwrap`, Landlock, Seatbelt); `workspaceRoot` nombra el directorio real del host en forma canónica de filesystem. La identidad del workspace se resuelve antes de la normalización léxica, así que un cwd válido que contenga `symlink/..` concede el directorio donde aterriza realmente `chdir`, no un padre léxico ajeno. Los contenedores, las microVMs y los ejecutores remotos NO son backends de este seam — reemplazan a los Service Providers de seams de capacidad enteros (`ctx.shell`, `ctx.fs`) como grupos coherentes de entorno. El límite y su justificación: [la Agent Note del sandbox](../../../.agents/notes/implemented/feature/2026-07-06-sandbox.md).

Implementaciones: [`@deepseek-ai/dsh-sandbox-local`](../sandbox-local/) (Linux: `bwrap`, si no, el lanzador Landlock por plataforma; macOS: `sandbox-exec`/Seatbelt). Consumidores: [`@deepseek-ai/dsh-bash-sandbox`](../../shell/bash-sandbox/) (envuelve `['bash', '-c', command]`).

## Experiencia del modelo

### Error de confinamiento, indirecto

#### Lo que ve el modelo

A través de [`dsh-bash-sandbox`](../../shell/bash-sandbox/README.md) y [`dsh-tool-bash`](../../shell/tool-bash/README.md), el fallo al cumplir un modo solicitado produce el código `SANDBOX_UNAVAILABLE` y el error exacto de abajo. Un fallo del runner en tiempo de ejecución añade ` Runner failure: <detail>`.

##### Error exacto

```markdown
sandbox mode "<mode>" is requested but no sandbox backend is usable on this host; refusing to run the command unconfined. Install bubblewrap or run a Landlock-enforcing kernel (Linux), ensure sandbox-exec is usable (macOS), or ensure the ACL restricted-token runner can start (Windows) — otherwise switch the consumer to danger-full-access.
```

#### Efecto en tokens

El texto de error condicional es visible para esa llamada y se conserva en el historial hasta la compactación.

#### Efecto en la caché KV

Solo append; el contenido recién visible sigue el prefijo de petición reutilizable y no invalida las entradas existentes de la caché KV.

## Limitaciones conocidas y trabajo diferido

- **Los efectos de archivo son todo el vocabulario de política** — el seam no expresa restricciones de red, proceso, syscall, dispositivo ni credenciales.
- **Confinamiento solo en el mismo mundo** — los contenedores, las microVMs y la ejecución remota exigen reemplazar implementaciones de capacidad, no añadir aquí un provider.
- **El reporte de denegación es un dialecto de stderr** — el seam devuelve firmas del backend en lugar de un canal de denegación tipado en runtime, así que los consumidores que necesitan clasificación deben inferirla de la salida del proceso hijo.
- **Los diagnósticos del runner son in-band** — el estado de salida más la evidencia de stderr no pueden probar qué proceso escribió una línea coincidente, así que un child confinado que imita deliberadamente a su runner puede provocar una atribución falsa de disponibilidad/diagnóstico. Esto no puede eludir el confinamiento; un canal de estado del runner out-of-band queda diferido.
- **Un provider por contexto** — componer mecanismos de sandbox distintos a la vez exige una escalera a nivel de provider o contextos de Cordis separados; los callers eligen política por llamada, no identidad de backend.
