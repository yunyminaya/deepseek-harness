# test-support/ — infraestructura de desarrollo y pruebas

[English](README.md) | Español

Estos paquetes dan soporte al desarrollo, los tests y los ejemplos del repositorio, no a las APIs de producto. Su compatibilidad sigue la necesidad de desarrollo que sirven.

| Paquete | Rol |
|---|---|
| [`acp-snapshot/`](acp-snapshot/README.es.md) | Proporciona el toolkit de tests de instantánea ACP |
| [`agent-loop-testkit/`](agent-loop-testkit/README.es.md) | Monta los prerrequisitos compartidos para los tests de AgentLoop |
| [`invariants/`](../runtime-diagnostics/invariants/README.es.md) | Ejecuta aserciones de contrato de runtime en tiempo de desarrollo |
| [`loader-smoke/`](loader-smoke/README.es.md) | Lanza aplicaciones compuestas por el Loader para pruebas de humo (smoke tests) |
| [`llm-mock-server/`](llm-mock-server/README.es.md) | Proporciona un servidor de fallos determinista compatible con OpenAI |
| [`llm-replay/`](llm-replay/README.es.md) | Reproduce respuestas de modelo grabadas para tests y demos sin clave |

Un paquete sale de `test-support/` cuando adquiere un contrato de producto y consumidores de producto.

El contrato de invariants se documenta en [docs/subsystems/invariants.md](../../docs/subsystems/invariants.es.md).
