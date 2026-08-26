# shell/ — familia de capacidad bash

[English](README.md) | Español

La familia de capacidad abarca el seam de ejecutor canónico, sus implementaciones, el entorno de shell compartido y las herramientas orientadas al modelo. Todos son paquetes **product**.

| Paquete | Rol | Clave ctx |
|---|---|---|
| [`shell/`](shell/README.es.md) | Define el contrato de ejecutor compartido por los Service Providers y los Consumers. | `ctx.shell` |
| [`bash-local/`](bash-local/README.es.md) | Ejecuta comandos a través del servicio local [`subprocess`](../subprocess/README.es.md). | (registra `ctx.shell`) |
| [`bash-sandbox/`](bash-sandbox/README.es.md) | Aplica el backend [`sandbox`](../sandbox/README.es.md) configurado antes de la ejecución local. | (registra `ctx.shell`) |
| [`pwsh-local/`](pwsh-local/README.es.md) | Ejecuta comandos de PowerShell con comportamiento de proceso específico de Windows. | (registra `ctx.shell`) |
| [`shell-env/`](shell-env/README.es.md) | Proporciona el entorno gestionado `DSH_*` compartido por las herramientas de shell. | `ctx.shellEnv` |
| [`tool-bash/`](tool-bash/README.es.md) | Expone al modelo la ejecución de Bash y la integración de jobs en segundo plano. | (se registra en `ctx.tools`) |
| [`tool-pwsh/`](tool-pwsh/README.es.md) | Expone al modelo la ejecución de PowerShell. | (se registra en `ctx.tools`) |

Un `cordis.yml` hoja selecciona una implementación de ejecutor y las herramientas orientadas al modelo que necesita. Una composición con sandbox selecciona además un provider de `ctx.sandbox`; el [ejemplo ACP](../../examples/acp-agent/) muestra un cableado completo.

La referencia del subsystem — vocabulario de petición/spec, resultados, procesos en segundo plano, el servicio y los eventos — es [docs/subsystems/shell.es.md](../../docs/subsystems/shell.es.md).
