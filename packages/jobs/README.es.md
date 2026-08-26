# jobs/ — familia de capacidades de trabajos en segundo plano

[English](README.md) | Español

Esta familia da a las herramientas de larga duración un único protocolo de trabajos en segundo plano aislado por propietario para la observación, la cancelación, la espera y los avisos de finalización.

| Paquete | Rol | clave ctx |
|---|---|---|
| [`jobs/`](jobs/README.es.md) | Define el registro de trabajos y el contrato de ciclo de vida | `ctx.jobs` |
| [`jobs-local/`](jobs-local/README.es.md) | Implementa el registro de trabajos local al proceso | se registra en `ctx.jobs` |
| [`tool-jobs/`](tool-jobs/README.es.md) | Expone el control de trabajos y los avisos de finalización al modelo | se registra en `ctx.tools` |

Ver las decisiones de [runtime de trabajos en segundo plano](../../.agents/notes/implemented/architecture/2026-06-20-generic-long-running-tool-runtime.es.md) y de [registro de trabajos](../../.agents/notes/implemented/architecture/2026-07-26-job-registry-seam.es.md).

La referencia de subsistema — el esquema de ids, el contrato con valla de propietario, las instantáneas — es [docs/subsystems/jobs.md](../../docs/subsystems/jobs.es.md); el diseño está en las Agent Notes de [runtime de trabajos en segundo plano](../../.agents/notes/implemented/architecture/2026-06-20-generic-long-running-tool-runtime.es.md) y de [contrato del registro de trabajos](../../.agents/notes/implemented/architecture/2026-07-26-job-registry-seam.es.md).
