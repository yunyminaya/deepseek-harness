# interaction/ — el plano de colaboración humana

[English](README.md) | Español

Los servicios y plugins a través de los cuales un humano colabora con un agent en ejecución — preguntas, aprobaciones, presets de permiso, comandos. Son paquetes de **producto**: interfaces reales que maneja una persona.

| Paquete | Rol | clave ctx |
|---|---|---|
| [`commands/`](commands/README.es.md) | Registra y despacha comandos humanos para los adaptadores interactivos. | `ctx.commands` |
| [`user-approval/`](user-approval/README.es.md) | Coordina las decisiones de aprobación de un solo uso. | `ctx.approval` |
| [`permission/`](permission-presets/README.es.md) | Presenta y persiste los presets de permiso visibles para el usuario. | `ctx.permissionPresets` |
| [`user-questions/`](user-questions/README.es.md) | Define el seam de pregunta/respuesta humana neutral respecto al provider. | `ctx.userQuestions` |
| [`tool-ask-user/`](tool-ask-user/README.es.md) | Expone las preguntas humanas al modelo. | (se registra en `ctx.tools`) |

Estos paquetes se integran a través de los contratos existentes de agent y de sesión en lugar de cambiar el loop. Las aplicaciones interactivas aportan los adaptadores concretos de comandos, aprobación y preguntas; la automatización usa [`acp/`](../acp/README.es.md), y los bundles de demo ejecutables viven en [`examples/`](../examples/README.es.md). La CLI de producto [`dsh`](../../apps/cli/README.es.md) compone estos paquetes directamente.

Las referencias de subsistema: [approval.md](../../docs/subsystems/approval.es.md), [permission-presets.md](../../docs/subsystems/permission-presets.es.md), [user-questions.md](../../docs/subsystems/user-questions.es.md) y [commands.md](../../docs/subsystems/commands.es.md). El transporte ACP solo de automatización es [`acp/`](../acp/README.es.md), la mitad de servidor JSON-RPC del SDK es [`sdk/server`](../sdk/README.es.md) y el pegamento de arranque de bin compartido es [`boot/`](../boot/README.es.md).
