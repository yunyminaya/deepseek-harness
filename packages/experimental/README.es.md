# experimental/ — paquetes experimentales privados

[English](README.md) | Español

Este grupo contiene prototipos y plugins de Cordis solo internos que usan el runtime real del repositorio sin unirse a una release oficial. Sus paquetes son privados, no ofrecen ninguna promesa de estabilidad ni soporte, y conservan los mismos requisitos de ingeniería, seguridad, documentación, ciclo de vida, tests e instantáneas que los paquetes de release.

| Paquete | Rol | clave ctx |
|---|---|---|
| `agent-team/` | Plantilla de Agent Teams de raíz implícita, buzón de pares persistente, DAG de tareas compartido y coordinación de runtime | `ctx.agentTeams` |
| `tool-agent-team/` | Herramientas de Agent Teams orientadas al modelo con ámbito y orientación de colaboración | — |

Las [reglas del subárbol](AGENTS.es.md) definen el aislamiento de dependencias, la exclusión de releases y la promoción.
