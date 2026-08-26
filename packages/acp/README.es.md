# acp/ — Automatización de Agent Client Protocol

[English](README.md) | Español

El grupo ACP expone los agents del harness a clientes programáticos a través del Agent Client Protocol. Es un transporte de interoperabilidad, no una capa de presentación ni de interacción humana; el *cliente* subagent fuera de proceso equivalente vive en [`subagent/subagent-acp`](../subagent/subagent-acp/README.es.md) porque implementa la interfaz de provider de subagent.

| Paquete | Rol |
|---|---|
| [`acp/`](acp/README.es.md) | Servidor ACP solo de automatización. |

El contrato del servidor está documentado en [`acp/README.md`](acp/README.es.md).
