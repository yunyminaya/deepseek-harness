# goal/ — objetivos persistentes de la misma sesión

[English](README.md) | Español

Estado de objetivo duradero para la sesión de un agent, propiedad independiente de las herramientas orientadas al modelo y de la política de continuación que lo consumen. El estado de objetivo forma parte del log de la sesión propietaria; los consumidores dependen de `dsh-goal`, nunca del agent loop concreto.

| Paquete | Rol | Clave ctx |
|---|---|---|
| [`goal/`](goal/README.es.md) | Estado de objetivo y ciclo de vida | `ctx.goals` |
| [`goal-round-driver/`](goal-round-driver/README.es.md) | Continuación de objetivo de la misma sesión | — |
| [`tool-goal/`](tool-goal/README.es.md) | Herramientas de objetivo orientadas al modelo | — |
| [`command-goal/`](command-goal/README.es.md) | Comando de objetivo orientado a humanos | — |

La referencia del subsistema —identidad de objetivo, instantáneas del ciclo de vida, activación, registros de cambio— está en [docs/subsystems/goal.md](../../docs/subsystems/goal.es.md).
