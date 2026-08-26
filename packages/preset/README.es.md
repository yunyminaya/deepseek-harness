# preset/ — composición de agent por sesión

[English](README.md) | Español

Un **agent preset** es un directorio que contiene un `agent.cordis.yml`. Montarlo bajo el contexto de ámbito de un agent da a esa sesión sus propias herramientas y secciones de prompt mientras cada otra sesión activa conserva las suyas, de modo que un solo proceso puede ejecutar a la vez varios agents compuestos de forma diferente.

| Paquete | Rol | clave de ctx |
|---|---|---|
| `agent-presets/` | Vocabulario de preset, descubrimiento de sistema de archivos sobre raíces confiables y de autoría de usuario, y el montaje por agent con salvaguardas | `ctx.agentPresets` |
| `persona/` | La persona del agent como fila componible, de modo que un preset pueda cambiar la identidad y no solo las herramientas | — |

Los presets que el despliegue entrega viven en [`apps/cli/config/agent-presets/`](../../apps/cli/config/agent-presets) — un directorio por preset, y ese listado de directorio es el roster. Nombrarlos también aquí sería una segunda lista que mantener sincronizada, y la primera en quedarse atrás.

La división de composición que asume este grupo: los registros y las instalaciones entre sesiones son singletons del proceso y permanecen en la composición del host, mientras que un preset lleva lo que un agent aporta a ellos. Un preset que nombra una fila que publica un servicio global del proceso se rechaza en el montaje en lugar de permitir que colisione con la siguiente sesión.

Diseño: [la Agent Note de agent presets por sesión](../../.agents/notes/implemented/architecture/2026-08-03-per-session-agent-presets.es.md).
