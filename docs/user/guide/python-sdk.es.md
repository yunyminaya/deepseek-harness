# Empieza con el SDK de Python

[English](python-sdk.md) | Español

Este tutorial es la alternativa programática a la interfaz web. Instala el SDK de Python publicado, ejecuta una composición de agent incluida en el repositorio y muestra cómo llamar a la misma API desde tu propio programa.

## Requisitos previos

- Python 3.10 o superior
- Git
- Linux x64, Linux arm64 o macOS 14 o superior en arm64
- Un endpoint de API compatible con DeepSeek y una credencial
- Un espacio de trabajo aislado que el agent pueda modificar

## Instala el SDK

Clona el repositorio para usar su ejemplo ejecutable, crea un entorno virtual e instala el SDK con su runtime incluido de la misma versión:

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
python -m venv .venv
. .venv/bin/activate
python -m pip install deepseek-harness-sdk
```

El runtime instalado no necesita un Node.js del sistema. Quienes contribuyan al repositorio y necesiten compilar el runtime o las wheels desde el código fuente deben usar los [flujos de trabajo de contribución de Python](../../../python/development.es.md).

## Ejecuta el ejemplo incluido

Fija la credencial en el entorno. Fija también `DEEPSEEK_BASE_URL` cuando el modelo lo sirva un proxy compatible con OpenAI en lugar del endpoint por defecto de DeepSeek.

```sh
export DEEPSEEK_API_KEY=sk-your-key-here
# export DEEPSEEK_BASE_URL=http://127.0.0.1:8000/v1
# export DSH_MODEL=deepseek-v4-flash
# export DSH_SYSTEM_PROMPT='You are a helpful software engineer assistant.'
```

Ejecuta una tarea contra un espacio de trabajo aislado y un directorio de sesiones:

```sh
python examples/jsonrpc-agent/minimal.py \
  --workspace /absolute/path/to/workspace \
  --session-root /absolute/path/to/sessions \
  --session-id example-001 \
  "Inspect the repository and fix the failing tests."
```

El script imprime la respuesta final del asistente. El directorio de sesiones recibe un log JSONL con las solicitudes de modelo ensambladas y las llamadas de herramienta.

## Usa el SDK en tu propio programa

El ejemplo incluido es un envoltorio fino alrededor de esta llamada al SDK:

```python
from pathlib import Path

from deepseek_harness import DeepSeekHarness

config = Path("examples/jsonrpc-agent/minimal.cordis.yml").resolve()
workspace = Path("/absolute/path/to/workspace").resolve()
sessions = Path("/absolute/path/to/sessions").resolve()

with DeepSeekHarness(
    provider="deepseek-official",
    model="deepseek-v4-flash",
    max_tokens=49_152,
    cwd=str(workspace),
    session_root=str(sessions),
    cordis=str(config),
) as harness:
    result = harness.run(
        "Inspect the repository and fix the failing tests.",
        session_id="example-001",
    )

print(result.final_response)
```

`DeepSeekHarness` inicia el runtime incluido de forma diferida y lo reutiliza hasta que el gestor de contexto sale. Reutilizar el mismo harness y el mismo id de sesión conserva el proceso Bash propiedad de la sesión, incluidos su directorio de trabajo, las variables exportadas y las funciones del shell. Usa un id de sesión nuevo para una tarea independiente; reutiliza un id solo cuando la siguiente llamada deba continuar la misma conversación duradera.

## Comprende la composición del ejemplo

| Propiedad | Valor |
|---|---|
| System prompt | `DSH_SYSTEM_PROMPT`, con `You are a helpful software engineer assistant.` como valor por defecto |
| Modelo en `minimal.py` | `--model`, luego `DSH_MODEL`, luego `deepseek-v4-flash` |
| Herramientas visibles para el modelo | Solo `bash` persistente y `str_replace_editor` |
| Tiempo de espera de Bash | 300 segundos |
| Límite de salida del editor | 16.000 caracteres |
| Compactación de contexto | Desactivada |
| Sistema de archivos | Backend local pelado; las rutas absolutas del editor pueden apuntar a cualquier ruta visible para el proceso del runtime |
| Persistencia de sesión | JSONL sin comprimir en `DSH_SESSION_ROOT` |

La composición omite la identidad del harness, el texto del prompt del espacio de trabajo, las skills, el Bash de un solo uso, las herramientas de tarea, la compactación y cualquier otro plugin visible para el modelo. Los hechos de política del sandbox se registran como contexto de usuario del runtime en lugar de añadirse al system prompt.

## Elige los IDs de espacio de trabajo y sesión

`cwd` selecciona el espacio de trabajo disponible para el agent, mientras que `session_root` guarda los logs y el estado de la sesión. Usa un id de sesión nuevo para una tarea independiente; reutiliza un id solo cuando la siguiente llamada deba continuar la misma conversación y el mismo estado de shell persistente.

La composición usa `danger-full-access`. Ejecútala solo dentro de un clon o contenedor desechable: Bash y el editor pueden modificar cualquier ruta permitida al proceso del runtime. El backend PTY persistente requiere un sustrato de terminal POSIX, así que esta composición no admite agents de Windows.

La [referencia del ejemplo `jsonrpc-agent`](../../../examples/jsonrpc-agent/README.es.md) es la dueña de la composición exacta. La [referencia del SDK de Python](../../../python/sdk/README.es.md) cubre el ciclo de vida, los resultados, las notificaciones, la selección del runtime y la configuración; el [primer de Cordis](../../cordis-primer.es.md) cubre la sintaxis de composición.
