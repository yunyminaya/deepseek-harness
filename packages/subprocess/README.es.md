# subprocess/ — familia de capacidad de subprocess

[English](README.md) | Español

El sustrato de procesos compartido de un único mundo de ejecución: búsqueda de ejecutables, árboles de procesos hijos gestionados completamente especificados con stdio crudo o recogido, y un único primitivo profundo de proceso de terminal que es dueño de la asignación de PTY, los grupos en primer plano y la limpieza de sesión observable por el provider. El establecimiento de valores por defecto de comandos, la semántica de shell, los plazos, el encuadre de protocolo, la disponibilidad y la presentación permanecen en los Consumers — los [ejecutores bash](../shell/README.es.md), el [host LSP](../lsp/README.es.md), el [backend de shell PTY](../terminal/README.es.md) y el [backend de subagente ACP](../subagent/README.es.md). Véase el [Agent Note del seam de subprocess](../../.agents/notes/implemented/architecture/2026-07-26-subprocess-seam.es.md).

| Paquete | clave ctx | Rol |
|---|---|---|
| [`subprocess`](subprocess/README.es.md) (`@deepseek-ai/dsh-subprocess`) | `ctx.subprocess` | Service Definition: búsqueda de ejecutables, spawns gestionados ordinarios, el primitivo de proceso de terminal, los ciclos de vida de los manejadores y el vocabulario compartido de entorno/salida |
| [`subprocess-local`](subprocess-local/README.es.md) (`@deepseek-ai/dsh-subprocess-local`) | — | Service Provider local: árboles de procesos desprendidos, recogida/spill acotados, `node-pty`, inspección de primer plano/sesión, señalización de árbol y liberación por terminar-y-unirse |

El servicio es dueño del ciclo de vida del proceso a través de las recargas del Consumer; los Consumers son dueños de lo que significa un proceso (un comando bash, un futuro ejecutor sin shell) y de cada valor por defecto que lo conforma.

La referencia de subsistema — specs de spawn, lectores de salida, resultados, el entorno `DSH_*` — es [docs/subsystems/subprocess.md](../../docs/subsystems/subprocess.es.md); la decisión del seam está en el [Agent Note del seam de subprocess](../../.agents/notes/implemented/architecture/2026-07-26-subprocess-seam.es.md).
