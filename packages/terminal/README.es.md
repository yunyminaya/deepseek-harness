# terminal/ — familia de capacidad PTY persistente

[English](README.md) | Español

`PTY` son las siglas de **Pseudo-Terminal**（伪终端）. Esta capacidad ofrece sesiones de terminal persistentes con ámbito de propietario para flujos de trabajo que requieren estado entre llamadas de herramienta o stdin interactivo. PTY complementa las herramientas bash y de sistema de archivos de un solo uso; no sustituye sus contratos por operación, más fuertes.

| Paquete | Rol | clave ctx |
|---|---|---|
| [`pty`](terminal/README.es.md) (`@deepseek-ai/dsh-terminal`) | Registro backend, ids de marca, propiedad exacta del Agent, operaciones de sesión y limpieza esperada | `ctx.terminals` |
| `terminal-bash` (`@deepseek-ai/dsh-terminal-bash`) | Backend de shell sobre `ctx.subprocess.spawnTerminal`: detección de disponibilidad, estado de terminal acotado, política de sandbox y operaciones de sesión | se registra en `ctx.terminals` |
| `tool-terminal` (`@deepseek-ai/dsh-tool-terminal`) | Seis herramientas orientadas al modelo e integración de tareas genérica para envíos en segundo plano | se registra en `ctx.tools` |

El diseño y los límites diferidos viven en el [Agent Note de PTY persistente](../../.agents/notes/implemented/feature/2026-07-16-persistent-pty-sessions.es.md).

La referencia de subsistema — ids, contratos backend/sesión, disponibilidad de envío, lecturas acotadas — es [docs/subsystems/terminal.md](../../docs/subsystems/terminal.es.md); el diseño y los límites diferidos están en el [Agent Note de PTY persistente](../../.agents/notes/implemented/feature/2026-07-16-persistent-pty-sessions.es.md).
