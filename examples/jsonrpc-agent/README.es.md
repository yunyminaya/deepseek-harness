# jsonrpc-agent

[English](README.md) | Español

La composición de coding agent sin supervisión para el runtime JSON-RPC incluido en el SDK de Python. Deliberadamente no carga ninguna UI de terminal, logger de consola, UI de aprobación ni herramienta de preguntas al usuario, porque stdout pertenece al protocolo del SDK y los turnos los conduce el SDK.

Las herramientas orientadas al modelo son:

- `bash`, solo en primer plano (foreground)
- `read`, `write` y `edit`
- `subagent`, con un provider de spawn en proceso y en primer plano
- `todo_write`

El runtime circundante también carga persistencia de sesiones JSONL y compactación automática de contexto. `maxTokensAsSuccess` conserva un turno de modelo limitado por tokens como resultado de evaluación aceptado, preservando su razón `max-tokens`.

## Entorno de runtime

| Variable | Propósito |
|---|---|
| `DEEPSEEK_API_KEY` | Credencial pasada al endpoint del host compatible con OpenAI |
| `DEEPSEEK_BASE_URL` | Endpoint del host usado por `dsh-llm-deepseek` |
| `DSH_CWD` | Workspace del agent para las herramientas de bash y sistema de archivos |
| `DSH_CONTEXT_WINDOW` | Capacidad de contexto registrada para la entrada del catálogo `DSH_MODEL` en la variante mínima |
| `DSH_MAX_TOKENS_AS_SUCCESS` | `true` (por defecto) acepta los resultados limitados por tokens; `false` los reporta como errores |
| `DSH_MODEL` | Modelo por defecto usado por `minimal.py`; `--model` tiene prioridad |
| `DSH_SESSION_ROOT` | Directorio de sesiones JSONL |
| `DSH_SYSTEM_PROMPT` | Persona de coding aportada por el despliegue |

Pasa la ruta de configuración mediante la opción `cordis` del SDK de Python o `DSH_CORDIS_CONFIG`. El ejecutable incluido ya lleva todos los plugins que nombra este archivo; la máquina de destino no necesita Node.js.

## Variante mínima

[`minimal.cordis.yml`](minimal.cordis.yml) es la contraparte independiente completa del preset `minimal` de Web. `DSH_SYSTEM_PROMPT` selecciona su system prompt, con `You are a helpful software engineer assistant.` como respaldo. Suprime toda contribución de contexto de runtime al system prompt para las sesiones nuevas y no monta ningún plugin de compactación de contexto. Sus herramientas orientadas al modelo son exactamente:

- `bash` persistente con ámbito del propietario
- `str_replace_editor` con `view`, `create`, `str_replace` e `insert`

Compone el PTY local, el backend `fs-local` pelado, la política danger-full-access para Bash persistente y la persistencia JSONL sin comprimir que necesita el runtime incluido. Bash y las rutas absolutas del editor pueden modificar cualquier ruta disponible para el proceso de runtime, así que ejecuta esta variante solo contra un checkout o contenedor desechable. El PTY persistente requiere un entorno de terminal POSIX y no es una interfaz de agent para Windows.

[`minimal.py`](minimal.py) ejecuta la composición a través del SDK de Python y usa `DSH_MODEL` como su modelo por defecto. El [tutorial del SDK de Python](../../docs/user/guide/python-sdk.es.md) cubre la instalación, la ejecución, la selección de workspace y la identidad de sesión; la [referencia del SDK](../../python/sdk/README.es.md) es dueña del ciclo de vida del runtime y de la semántica de resultados.
