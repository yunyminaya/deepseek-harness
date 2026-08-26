# e2b/ — familia de runtime remoto E2B

[English](README.md) | Español

Un POC experimental de composición de providers que sitúa un mundo de ejecución de sistema de archivos/procesos en un sandbox Linux de E2B. E2B aporta solo el ciclo de vida del sandbox y los dos adaptadores fundamentales del SO; los consumers neutrales frente al provider construyen capacidades superiores sobre ellos.

| Paquete | Clave ctx | Rol |
|---|---|---|
| [`e2b`](e2b/README.es.md) (`@deepseek-ai/dsh-e2b`) | `ctx.e2b` | Crear un sandbox, preparar sus directorios de trabajo/runtime, exponer el handle compartido del SDK y eliminarlo al expirar el tiempo o al eliminarse |
| [`fs-e2b`](fs-e2b/README.es.md) (`@deepseek-ai/dsh-fs-e2b`) | `ctx.fs` | Implementar el seam del sistema de archivos sobre las APIs de E2B Filesystem |
| [`subprocess-e2b`](subprocess-e2b/README.es.md) (`@deepseek-ai/dsh-subprocess-e2b`) | `ctx.subprocess` | Implementar la búsqueda de ejecutables, los grupos de procesos gestionados y stdio, los archivos de spill remotos y las sesiones de terminal sobre las APIs de E2B Commands y PTY |

Los existentes [`dsh-bash-local`](../shell/bash-local/README.es.md), [`dsh-terminal-bash`](../terminal/terminal-bash/README.es.md) y [`dsh-lsp-stdio`](../lsp/lsp-stdio/README.es.md) no necesitan forks específicos de E2B. Delegan cada operación del mundo de ejecución en `ctx.fs` y `ctx.subprocess`, así que montar los dos adaptadores E2B sitúa su trabajo mutable en el mismo sandbox.

Este límite no mueve el proceso del harness, los objetos de Cordis, las llamadas al modelo, el estado agent/sesión, la persistencia de sesión, los skills, el estado de protocolo de nivel superior ni los buffers del SDK de E2B. La [decisión del mundo de ejecución portable](../../.agents/notes/implemented/architecture/2026-07-28-portable-execution-world-consumers.es.md) es dueña tanto de la composición genérica como de este límite del POC.
