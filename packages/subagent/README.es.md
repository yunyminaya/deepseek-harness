# subagent/ — familia de capacidad de subagentes

[English](README.md) | Español

Esta familia permite que un agent (agente) delegue trabajo a agentes hijos. Varios providers con nombre pueden coexistir en un mismo contexto.

| Paquete | Rol | clave ctx |
|---|---|---|
| [`subagent/`](subagent/README.md.es.md) | Define el registro de providers, la delegación y la continuación | `ctx.subagents` |
| [`subagent-inprocess/`](subagent-in-process-driver/README.md.es.md) | Proporciona el driver de ejecución en proceso compartido | — |
| [`subagent-spawn-in-process/`](subagent-spawn-in-process/README.md.es.md) | Inicia un hijo en proceso completamente nuevo | se registra en `ctx.subagents` |
| [`subagent-fork-in-process/`](subagent-fork-in-process/README.md.es.md) | Inicia un hijo en proceso a partir del historial completado del padre | se registra en `ctx.subagents` |
| [`subagent-acp/`](subagent-acp/README.md.es.md) | Inicia un hijo fuera de proceso sobre ACP | se registra en `ctx.subagents` |
| [`subagent-codex/`](subagent-codex/README.md.es.md) | Inicia un hijo real de Codex app-server | se registra en `ctx.subagents` |
| [`subagent-claude-code/`](subagent-claude-code/README.md.es.md) | Inicia un hijo real de Claude Code mediante el SDK oficial de Claude Agent | se registra en `ctx.subagents` |
| [`subagent-dsh-sdk/`](subagent-dsh-sdk/README.md.es.md) | Inicia un hijo de Harness fuera de proceso mediante el SDK de TypeScript | se registra en `ctx.subagents` |
| [`tool-subagent/`](tool-subagent/README.md.es.md) | Expone la delegación al modelo | se registra en `ctx.tools` |
| [`tool-subagent-control/`](tool-subagent-control/README.md.es.md) | Expone al modelo el envío de mensajes y el listado de hijos | se registra en `ctx.tools` |
| [`tool-subagent-report/`](tool-subagent-report/README.md.es.md) | Proporciona el canal de informe hijo→padre | se registra en los ámbitos hijos |

Los paquetes de Codex y Claude Code son Profile Bundles opcionales independientes. Instala uno o ambos con `dsh plugin --profile <name> add @deepseek-ai/dsh-subagent-codex @deepseek-ai/dsh-subagent-claude-code`, y reinicia ese Profile después; cada paquete registra solo su provider Host inactivo. Para conceder una herramienta, copia un Agent Preset completo, elimina `disabled` de cada fila de herramienta coincidente e inicia una nueva sesión. Eliminar un paquete retira solo ese provider y su clausura de runtime privada en el siguiente arranque del Profile.

Consulta las decisiones de la [familia de capacidad](../../.agents/notes/implemented/feature/2026-06-21-subagent-capability-seam.es.md), los [hijos continuables](../../.agents/notes/implemented/feature/2026-07-21-continuable-background-subagents.es.md) y las [herramientas de control](../../.agents/notes/implemented/simplification/2026-07-26-merge-subagent-control-service.es.md).

La referencia del subsistema — solicitudes de inicio, resultados, ejecuciones en vivo, el contrato del provider, los hijos en segundo plano continuables — está en [docs/subsystems/subagent.md](../../docs/subsystems/subagent.es.md); el fundamento de diseño en las Agent Notes [seam de capacidad de subagente](../../.agents/notes/implemented/feature/2026-06-21-subagent-capability-seam.es.md), [subagentes (subagent) en segundo plano continuables](../../.agents/notes/implemented/feature/2026-07-21-continuable-background-subagents.es.md) y [servicio de control de subagentes fusionado](../../.agents/notes/implemented/simplification/2026-07-26-merge-subagent-control-service.es.md).
