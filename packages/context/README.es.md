# context/ — extensiones de contexto de petición

[English](README.md) | Español

Plugins de producto que añaden contexto de petición visible para el modelo sin definir una herramienta. `agent-instructions` se incluye en el bundle por defecto `dsh-agent-spine-demo` y se puede desactivar mediante la configuración del bundle; `time-context`, `tmux-context`, `session-reference`, `file-reference` y `file-reference-local` son opcionales.

| Paquete | Rol | Clave de ctx |
|---|---|---|
| [`session-reference/`](session-reference/README.es.md) | Instantáneas acotadas de otras sesiones | `ctx.sessionReferenceResolver` |
| [`file-reference/`](file-reference/README.es.md) | Seam de descubrimiento de referencias a archivos y gramática `@file` | `ctx.fileReferences` |
| [`file-reference-local/`](file-reference-local/README.es.md) | Provider de referencias a archivos en el sistema de archivos local | — |
| [`time-context/`](time-context/README.es.md) | Contexto de hora actual y tiempo transcurrido | — |
| [`tmux-context/`](tmux-context/README.es.md) | Contexto de ubicación de tmux | — |
| [`agent-instructions/`](agent-instructions/README.es.md) | Contexto de instrucciones de workspace | — |

Las referencias a sesiones se documentan en [docs/subsystems/session-reference.md](../../docs/subsystems/session-reference.es.md); el [registro de decisión de `agent-instructions`](../../.agents/notes/implemented/feature/2026-06-24-workspace-context.es.md) es el dueño de su aislamiento por agent/sesión y de su división de ciclo de vida.
