# workflow/ — familia de la capacidad dynamic-workflow

[English](README.md) | Español

Esta familia ejecuta flujos de trabajo de orquestación creados por el modelo sobre subagentes (subagents) y expone al modelo herramientas generales y de política fija.

| Paquete | Rol | clave ctx |
|---|---|---|
| [`workflow/`](workflow/README.es.md) | Define la ejecución de flujos de trabajo y los eventos de su ciclo de vida | `ctx.workflowEngine` |
| [`workflow-worker-thread/`](workflow-worker-thread/README.es.md) | Ejecuta los scripts de flujo de trabajo en hilos de trabajo | se registra en `ctx.workflowEngine` |
| [`tool-workflow/`](tool-workflow/README.es.md) | Expone al modelo la ejecución general de flujos de trabajo | se registra en `ctx.tools` |
| [`tool-ralph/`](tool-ralph/README.es.md) | Expone el flujo de trabajo Ralph fijo de agent nuevo | se registra en `ctx.tools` |

Los hilos de trabajo aíslan la ejecución del flujo de trabajo del event loop del host, pero no constituyen una frontera de seguridad. Ver las decisiones [dynamic-workflow](../../.agents/notes/implemented/feature/2026-07-05-dynamic-workflows.es.md) y [herramienta Ralph](../../.agents/notes/implemented/feature/2026-07-19-fresh-agent-ralph-workflow-tool.es.md).

La referencia del subsistema — peticiones de inicio, `WorkflowMeta`, resultados, ejecuciones en curso, eventos `workflow/*` — es [docs/subsystems/workflow.md](../../docs/subsystems/workflow.es.md); las decisiones, en los Agent Notes [dynamic-workflows](../../.agents/notes/implemented/feature/2026-07-05-dynamic-workflows.es.md) y [Consumer Ralph](../../.agents/notes/implemented/feature/2026-07-19-fresh-agent-ralph-workflow-tool.es.md).
