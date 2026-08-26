# @deepseek-ai/dsh-agent-spine-demo

[English](README.md) | Español

El **spine de agent (agente) por defecto, sin ejecutor y sin interfaz de usuario**, como UN solo plugin de bundle de Cordis. Carga el conjunto fijo de servicios que necesita todo agent del harness (marco de trabajo para agentes), incluido el skill (destreza) provider local, y reenvía la lista `agents` del agent loop (bucle del agente) como configuración propia — de modo que un paquete de aplicación compone un agent funcional añadiendo solo un punto de entrada y los backends intercambiables.

Lee este paquete para ver el árbol de plugins completo y su orden de composición.

## El árbol que carga

`apply(ctx, config)` monta cada uno de estos como hijo del fiber del bundle:

```
@deepseek-ai/cordis-plugin-timer  timer service (writes nothing to stdout)
@deepseek-ai/dsh-llm              abstract LLM service + content-block vocabulary
@deepseek-ai/dsh-session          event-sourced session log + store
@deepseek-ai/dsh-session-title    log-backed title service + deterministic fallback
@deepseek-ai/dsh-system-prompt    prompt-section + tool-schema assembly
@deepseek-ai/dsh-tools            registry + guarded pre/around/post/final-result pipeline
@deepseek-ai/dsh-skill            skill provider registry
@deepseek-ai/dsh-skill-filesystem      local filesystem skill provider
@deepseek-ai/dsh-agent            agent registry + initiator scope + agent/* events
@deepseek-ai/dsh-goal             optional persisted same-session goal domain
@deepseek-ai/dsh-tool-goal        optional model-facing goal controls
@deepseek-ai/dsh-goal-round-driver     optional same-session goal-round driver
@deepseek-ai/dsh-llm-retry        provider-routed request retry policy
@deepseek-ai/dsh-jobs-local      generic background-job registry
@deepseek-ai/dsh-invariants       configurable invariant registry service
@deepseek-ai/dsh-session/invariant
@deepseek-ai/dsh-agent/invariant
@deepseek-ai/dsh-scope/invariant
@deepseek-ai/dsh-agent-loop/invariant
                                  package-owned relational checks
@deepseek-ai/dsh-tool-bash        the model-facing bash schema (unless toolBash=false)
@deepseek-ai/dsh-agent-instructions  AGENTS.md/CLAUDE.md workspace context loader
@deepseek-ai/dsh-tool-skill       session-prefix skill catalog + model-facing loader schema
@deepseek-ai/dsh-tool-jobs       job_output/job_list/job_kill schemas + completion notices
@deepseek-ai/dsh-agent-loop       THE concrete loop (gets the forwarded `agents`)
                                  (dsh-system-prompt gets the forwarded `persona`)
```

## Lo que deja deliberadamente FUERA del bundle

El spine es todo lo COMÚN a cada punto de entrada. Las piezas intercambiables y acopladas al punto de entrada quedan fuera, elegidas por quien cargue el bundle:

- **el adaptador de LLM** — el bundle trae el servicio `llm` abstracto; la hoja registra un adaptador concreto en `ctx.llm` (`llm-deepseek`, `llm-pi-ai`, `llm-replay`).
- **los providers de título de sesión respaldados por modelo** — el bundle monta el servicio de respaldo con límites de ejemplo sustituibles (5 palabras, 40 bytes de respaldo, 80 bytes de título aceptado); una hoja puede optar por exactamente un provider de LLM de primer prompt o de todos los mensajes.
- **el ejecutor de bash** — el bundle trae `tool-bash` (el schema del consumidor); la hoja aporta `ctx.shell` (`bash-local` o una impl en sandbox).
- **los skill providers no locales** — el bundle trae el registro de skills, el provider local de sistema de archivos y la herramienta `skill`; los despliegues pueden añadir otros providers, como catálogos embebidos o remotos, como hermanos.
- **punto de entrada + infraestructura por aplicación** — los paquetes de aplicación headless, ACP y JSON-RPC son dueños del transporte, de stdout y de las decisiones de recarga. `timer` permanece en el spine porque es común y silencioso en stdout.

Esto aplica la separación [Service Definition / Service Provider / Consumer](../../../.agents/notes/implemented/architecture/2026-06-13-capability-seams.md) a nivel de composición: el bundle es dueño del spine compartido, la hoja es dueña de los backends, y el paquete de aplicación es dueño del punto de entrada.

## Config

```ts
import type { Config } from '@deepseek-ai/dsh-agent-spine-demo'
// { agents?, maxParallelToolCalls?, includeHarnessIdentity?, includeRuntimeContext?, persona?, toolOrder?, tools?, dshHome?, sessionTitle?, skills?, workspaceContext, toolBash?, jobs?, toolJobs?, goals?, invariants? }
// workspaceContext requires { maxBytes } or false; the other owner schemas supply defaults.
```

El bundle reenvía cada campo al hijo que es dueño de él. Los paquetes de aplicación aportan los agents precreados: las composiciones headless y JSON-RPC crean `main`, mientras que la app ACP crea agents bajo demanda en `session/new`. `includeRuntimeContext: false` se reenvía a `dsh-system-prompt` y suprime todas las instantáneas de contexto dinámico para las sesiones nuevas sin deshabilitar sus servicios de política. Los ajustes de prompt, herramienta, título, skill, instrucciones de agent, invariant, goal y tarea conservan los schemas y valores por defecto que documentan sus paquetes propietarios; `jobs.maxConcurrentJobsPerOwner` configura el provider local de forma independiente de los controles `toolJobs` orientados al modelo. `pickSpineConfig()` copia solo los campos propiedad de este bundle, y los valores `dshHome` en conflicto fallan durante la composición.

Por ejemplo, `{ invariants: { enabled: true, package_allowlist: ['^@deepseek-ai/dsh-'], package_blocklist: ['agent-loop$'] } }` mantiene montados los compañeros propiedad del paquete pero suprime el propietario bloqueado. Las coincidencias de la lista de bloqueados anulan las de la lista de permitidos; consulta [`dsh-invariants`](../../runtime-diagnostics/invariants/README.es.md) para las reglas de regex y ciclo de vida.

## Por qué un bundle de código y no un include YAML compartido

Un include YAML puede deduplicar configuración, pero no puede ser dueño de un bin ni aportar valores por defecto al punto de entrada. El paquete de aplicación ACP convierte el cableado de stdout puro de protocolo en el valor por defecto, aunque una hoja aún puede añadir un logger inseguro. Los hijos del bundle registran servicios en el almacén de la raíz con clave de isolate, de modo que los hermanos de hoja inyectados los ven sin acoplamiento de orden de carga.

La política de reintentos puede repetir una solicitud fallida en un nuevo paso numerado. El estado del reintento, los errores del provider y los trozos parciales fallidos quedan fuera del historial del modelo; cada intento del provider aún puede generar cobros, el modo `always` no tiene límite de intentos, los puntos de entrada derivan el uso a través de cada paso registrado, y la solicitud reconstruida conserva el prefijo anterior para la reutilización de la caché del provider.

## Experiencia del modelo

Indirectamente, a través de `dsh-system-prompt`, `dsh-tool-skill`, `dsh-tool-bash`, `dsh-tools` y `dsh-llm-retry`, más `dsh-tool-goal` y los prompts de ronda de goals cuando `goals` está habilitado. El bundle no añade ningún contenido de envoltura vinculado al modelo propio.

#### Efecto de KV Cache

Sin invalidación directa; el consumidor nombrado es dueño de cualquier cambio de prefijo de petición.

## Limitaciones conocidas y trabajo aplazado

- **La mayor parte del conjunto del spine está fija en código** — `apply()` siempre monta los servicios del núcleo; la configuración puede omitir goals, skills, bash y las herramientas de control de tareas del bundle, pero cambiar el loop o descartar otro miembro del spine implica componer un bundle distinto.
- **El servicio de invariants y sus compañeros siguen siendo miembros fijos** — `invariants.enabled: false` o los filtros de paquetes suprimen las comprobaciones pero no eliminan los registros del servicio ni de los compañeros; la validación y congelación siempre activas de Session son cosas aparte.
