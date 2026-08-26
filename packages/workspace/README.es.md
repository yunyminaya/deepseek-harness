# workspace/ — familia de entidades workspace

[English](README.md) | Español

Esta familia es dueña de los workspaces persistentes: directorios de usuario con títulos y membresía ordenada de sesiones.

| Paquete | Rol | clave ctx |
|---|---|---|
| [`workspace/`](workspace/README.es.md) | Registra workspaces y la pertenencia de sus sesiones | `ctx.workspaceRegistry` |

La [referencia del paquete workspace](workspace/README.es.md) es dueña del ciclo de vida, la persistencia y las semánticas de eliminación.

La referencia del subsistema — la entidad, el canon de realpath, registro/resolución — es [docs/subsystems/workspace.md](../../docs/subsystems/workspace.es.md); el diseño de almacenamiento, en el [Agent Note de almacenamiento KV de dominio](../../.agents/notes/proposed/architecture/2026-07-24-domain-kv-storage-and-workspace.es.md).
