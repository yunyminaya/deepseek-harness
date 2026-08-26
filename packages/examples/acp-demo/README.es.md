# @deepseek-ai/dsh-acp-demo

[English](README.md) | Español

App de servidor de automatización ACP: el agent spine por defecto, agents creados por el cliente a través de [`@deepseek-ai/dsh-acp`](../../acp/acp/README.es.md), persistencia JSONL y checkpointing semántico detrás de un único bin JSON-RPC sobre stdio. Los clientes programáticos crean sesiones nuevas; este paquete no monta ninguna UI humana.

## Composición

| Plugin | Rol |
|---|---|
| `@deepseek-ai/dsh-agent-spine-demo` | Agent spine sin providers y sin agents precreados; `session/new` crea cada agent. |
| `@deepseek-ai/dsh-session-persistence-jsonl` | Registros de sesión durables usados por el checkpointing, la observabilidad y la reproducción de instantáneas. |
| `@deepseek-ai/dsh-session-checkpoint-policy` | Barreras de durabilidad antes de las llamadas al modelo y de los efectos de herramientas de nivel superior, más checkpoints de pasos completados. |
| `@deepseek-ai/dsh-session-query-sqlite` | Servicio derivado de consulta de sesión exacta/FTS, abierto antes del transporte ACP para que los consumers de hoja estén listos para la primera petición al modelo. |
| `@deepseek-ai/dsh-acp` | Transporte ACP solo de automatización sobre stdin/stdout. |

La app no instala comandos, interacción de usuario, navegación de sesión, selectores de configuración ni un logger de stdout. Posee estos plugins mediante un único efecto ordenado, de modo que el servicio de consulta esté listo antes de que ACP acepte trabajo y las sesiones ACP entren en quiescencia antes de que el checkpointing y la persistencia se separen. Las configuraciones de hoja aportan los plugins de LLM, executor, sandbox, aprobación, sistema de archivos y de herramientas orientadas al modelo.

## Config

| Clave | Valor por defecto | Se enruta a |
|---|---|---|
| `provider` | requerido | Ruta de provider para cada agent creado por ACP. |
| `model` | requerido | Modelo para cada agent creado por ACP. |
| `maxParallelToolCalls` | valor por defecto de agent-loop | Tope de concurrencia de llamadas de herramienta en enteros positivos; `1` es serial. |
| `persona` | — | Plantilla de persona de despliegue para `dsh-system-prompt`. |
| `toolOrder` | lexicográfico | Orden explícito de herramientas orientadas al modelo para `dsh-system-prompt`. |
| `tools` | `{ mode: 'native' }` | Transporte de herramientas del modelo nativo, Code Mode o combinado. |
| `dshHome` | `$DSH_HOME` o `~/.dsh` | Home del harness compartido por bash y el descubrimiento local de skills. |
| `sessionTitle` | límites del ejemplo de spine | Límites durables del título de respaldo; los títulos permanecen fuera del cable ACP. |
| `persistenceRoot` | `./.sessions` | Raíz del backend JSONL y directorio padre del índice derivado `session-query.db`. |
| `packChunks` | `true` | Empaquetar en el almacenamiento los eventos consecutivos de fragmentos delta. |
| `persistenceCompression` | `zstd` | Tramas Zstandard con checksum o `none` crudo. |
| `workspaceContext` | requerido | Presupuesto de bytes/configuración de instrucciones de workspace, o `false`. |
| `skills` | valores por defecto del propietario | Registro de skills, provider local y herramienta de skill orientada al modelo. |
| `toolBash` | valores por defecto del propietario | Config de la herramienta bash orientada al modelo. |
| `jobs` | `{ maxConcurrentJobsPerOwner: 10 }` | Admisión de tareas activas por propietario, local al proceso. |
| `toolJobs` | valores por defecto del propietario | Config genérica de control de trabajos en segundo plano, o `false`. |
| `goals` | valores por defecto del propietario | Dominio de objetivos de la misma sesión persistido y herramientas del modelo, o `false`. |

El [`examples/acp-agent/cordis.yml`](../../../examples/acp-agent/cordis.yml) incluido añade el adaptador de DeepSeek, providers de bash y de sistema de archivos con sandbox, política de aprobación única, compactación, subagents, flujos de trabajo, hooks y herramientas orientadas al modelo. La app aporta el índice derivado de consulta de sesión, mientras que el consumer de consulta orientado al modelo sigue siendo un opt-in explícito de la hoja. Los overlays de instantánea sustituyen solo los providers no deterministas o los valores de política.

## Bin

`dsh-acp-demo [--config path-to-cordis.yml]` (forma corta `-c`; por defecto `./cordis.yml`) carga el `.env` ignorado por git, salvo en modo replay; `DSH_SNAPSHOT=replay` selecciona el `cordis.snapshot.yml` hermano; el EOF de stdin elimina el contexto y vacía las sesiones antes de salir. El peer opcional instalado del Loader, `node-addon-require-builtin`, resuelve los especificadores pelados de plugins para el bin compilado bajo Node plano. Los diagnósticos usan stderr porque stdout es el cable ACP.

## Experiencia del modelo

Indirectamente, a través de `dsh-agent-spine-demo` y de los plugins de hoja orientados al modelo. El texto de prompt de ACP se convierte en el mensaje de usuario registrado ordinario; los metadatos de protocolo y las elecciones de permisos no entran en la petición al modelo.

#### Efecto en la KV Cache

Solo de añadido por sesión; la app no añade contenido al prefijo de petición por sí misma.

## Limitaciones conocidas y trabajo pendiente

- **La persistencia JSONL es fija** — otro backend exige otra composición.
- **Los plugins hermanos pueden corromper stdout** — la app no puede impedir que otra entrada escriba bytes no de protocolo.
- **Solo sesiones de automatización nuevas** — reanudar y la interacción humana pertenecen a otros puntos de entrada.
