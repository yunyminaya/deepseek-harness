# dsh-sandbox-policy — el hogar de la política de sandbox (`ctx.sandboxPolicy`)

[English](README.md) | Español

El único dueño de la resolución de la política de sandbox: el [`SandboxMode`](../sandbox/README.es.md) por defecto del despliegue y la raíz de respaldo, más la anulación de modo durable de cada sesión y la raíz de workspace inmutable. Cada capacidad que aplica enforcement recibe una política resuelta de modo-y-raíz por llamada; antes de cada petición, el modelo recibe la política actual sin un inventario de capacidades aparte.

## Por qué un hogar compartido

Las herramientas del sistema de archivos, los comandos bash one-shot y las sesiones de terminal pueden aplicar el mismo vocabulario de modos en combinaciones distintas. Si cada una resolviera su propio `mode` + `workspaceRoot`, podrían derivar hacia un mundo dividido, exactamente lo que advierte [la Agent Note de sandbox](../../../.agents/notes/implemented/feature/2026-07-06-sandbox.es.md). Cada backend que aplica enforcement consume la política completa resuelta por el dueño, mientras que el contexto actual describe solo qué significa esa política para cualquier operación disponible que aplique el sandbox de archivos de DSH. La [Agent Note de sandbox de fs entre familias](../../../.agents/notes/implemented/feature/2026-07-14-cross-family-fs-sandbox.es.md) registra la decisión de política compartida.

## Config

- `mode` — el `SandboxMode` por defecto del despliegue (`read-only` / `workspace-write` / `danger-full-access`), validado al cargar. Por defecto `read-only` (fail-safe).
- `workspaceRoot` — el directorio de respaldo bajo el que `workspace-write` puede escribir para llamadas sin agent o sesiones sin cwd. Por defecto `process.cwd()`, resuelto de todos modos a su identidad de sistema de archivos absoluta. Una llamada de agent normal usa el `cwd` inmutable de su cabecera de sesión.

## API

- `ctx.sandboxPolicy.resolve({ session?, mode? })` — resuelve una política completa por llamada. Un modo aprobado explícito supera al último evento `sandbox/mode` de la sesión, que a su vez supera a `defaultMode`; el `cwd` inmutable de la sesión se canoniza con semántica de sistema de archivos antes de convertirse en `workspaceRoot`; en caso contrario se aplica el respaldo configurado. La canonización precede a la normalización léxica para que `symlink/..` concuerde con la resolución del directorio de trabajo del proceso.
- `ctx.sandboxPolicy.defaultMode` / `ctx.sandboxPolicy.workspaceRoot` — el valor por defecto del despliegue y la raíz de respaldo que usa `resolve()`.
- `sandbox:policy` — una contribución de contexto segura para caché en el momento de la petición, derivada directamente de `resolve({ session })`. Enuncia el contrato de efecto de archivo neutro en capacidad del modo y el workspace canónico de la sesión bajo `workspace-write`; los dueños de herramientas conservan la guía de denegación y escalada específica de cada operación.
- `effectiveSandboxMode(events)` — el fold puro de los eventos `sandbox/mode` de una sesión (gana el último cambio, o `undefined`), usado dentro de `resolve()`.
- `setSandboxMode(session, mode)` — EL camino de escritura para una anulación por sesión: añade exactamente un evento `sandbox/mode`. El cambio ES su evento; nada muta el modo fuera de banda.
- `SANDBOX_MODES` — todos los modos, para la publicidad de opciones y la validación de runtime.

El companion `./invariant` opcional rechaza un evento durable `sandbox/mode` falsificado cuyo valor cae fuera de ese vocabulario cerrado; Session y su companion son dueños del almacenamiento circundante y de las reglas centrales de cerramiento de ejecución. El agent loop registra la instantánea completa de runtime-context ensamblada como un `user/message` con fuente, así que la entrada exacta de la política sigue siendo reconstruible sin un espejo en memoria de «lo último dicho».

## El almacén por sesión

Un cambio de runtime es un único evento `sandbox/mode` solo de log en la sesión a la que se aplica. `effective = explicit grant ?? fold(events) ?? deployment default`, así que una anulación sobrevive al reinicio mediante replay y dos sesiones nunca ven el estado de la otra. La identidad del workspace no necesita otro evento: el `SessionHeader.cwd` inmutable registrado en la creación es la raíz de cada llamada de esa sesión. El evento sigue siendo solo de log; antes de la siguiente petición, el dueño contribuye el hecho actual a la instantánea completa de runtime-context.

## Model Experience

### Política actual de sandbox de archivos

#### Qué ve el modelo

Una contribución `sandbox:policy` en la instantánea actual de runtime-context de cada sesión de agent. No enumera las capacidades montadas. Los plugins de herramientas conservan la guía de operación y escalada, la approval policy contribuye por separado a la misma instantánea y la guía de plan sigue siendo la sección de sistema de `dsh-plan-mode`.

##### Read-only

```markdown
Current DSH file policy: read-only. Any available operation enforced by the DSH file sandbox cannot modify files in the standing mode. Do not refuse a required modification from this policy alone: try an available tool normally and follow any denial and escalation guidance it returns.
```

##### Workspace-write

```markdown
Current DSH file policy: workspace-write. Any available operation enforced by the DSH file sandbox may modify files under the session workspace: "<workspace root>". Some platform temporary areas may also be writable.
```

##### Danger-full-access

```markdown
Current DSH file policy: danger-full-access. The DSH file sandbox does not restrict file modifications by available operations.
```

#### Efecto de tokens

Un mensaje de contexto durable y conciso en la primera petición y en cada cambio efectivo de política; las peticiones sin cambios no añaden nada. `workspace-write` lleva solo la ruta canónica del workspace de la sesión; las rutas temporales específicas de plataforma se resumen sin añadir bytes dependientes del host.

#### Efecto de KV Cache

El prompt de sistema estable permanece idéntico byte a byte entre cambios de modo. Una instantánea completa de contexto cambiada se anexa tras el historial retenido, preservando el prefijo cacheado anterior; las peticiones posteriores sin cambios reutilizan esa instantánea retenida.

## Limitaciones conocidas y trabajo aplazado

- **Una raíz de workspace principal por sesión** — la política resuelve `SessionHeader.cwd`; las raíces escribibles adicionales no forman parte de `SandboxExecutionPolicy`.
- **Solo modos de efecto de archivo** — `SandboxMode` gobierna los efectos de archivo; la política de red y de procesos queda fuera de su vocabulario, así que ningún control de aquí los restringe.
- **Las áreas temporales se resumen deliberadamente** — los backends con enforcement conceden distintas áreas temporales de plataforma, que se seleccionan después de la resolución de la política y, por tanto, no pueden enumerarse con veracidad en el contexto actual.
