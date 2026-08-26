# Agent Note: stdin + env adicional en el seam de bash

Status: implemented

[English](2026-06-30-bash-stdin-env-trusted-plugin-api.md) | Español

## Problema

El subsistema de hooks ejecuta comandos de hook externos como hacen Claude Code y Codex: un hook es un comando de shell que recibe su carga útil de evento como **JSON por stdin** y lee el contexto de un puñado de **variables de entorno** (`CLAUDE_PROJECT_DIR`, `CLAUDE_PLUGIN_ROOT`, `PLUGIN_ROOT`, …). El harness ya tiene un ejecutor de comandos perfectamente válido detrás del seam de la capacidad `ctx.shell` ([dsh-shell](../../../../packages/shell/shell) → [dsh-bash-local](../../../../packages/shell/bash-local)), con kills de grupo de procesos, truncado/derrame de salida y un saneamiento de credenciales. Reutilizarlo para la ejecución de hooks significa que un puente de hooks no reimplementa la fontanería de subprocesos — pero el seam no tenía forma de escribir stdin ni de fijar env adicional. Este cambio añade esas dos entradas.

`stdin` y `env` no crean una nueva capacidad de modelo porque la sintaxis ordinaria de shell ya aporta ambas. Las credenciales ambientales están protegidas por el saneamiento del entorno hijo de `dsh-bash-local`, no ocultando estos campos de Service Definition; los argumentos de las herramientas del modelo son JSON estático y no expanden variables de shell. Por tanto, los campos sirven a llamadores confiables dentro del proceso, como los puentes de hooks, que necesitan pasar entrada estructurada y variables `CLAUDE_*` sin incrustarlas en texto de shell visible para el modelo. Consulta [defensive-patterns.md](../../../../docs/defensive-patterns.es.md) para la regla del entorno ambiental.

## Decisión

Añade `stdin?: string` y `env?: Record<string, string>` a **ambos**, `ShellExecRequest` (la solicitud orientada al modelo/plugin) y `ShellExecSpec` (el spec resuelto sobre el que actúan `run`/`start`), y pásalos a través de `dsh-bash-local`: `resolve()` los transporta verbatim, `run()`/`start()` los pasan a `runBash`, que escribe los bytes en el stdin del hijo y fusiona el env adicional.

Tres decisiones deliberadas:

1. **La herramienta orientada al modelo omite `stdin` y `env`.** La sintaxis de shell ya cubre esas necesidades, así que unos parámetros duplicados añadirían superficie sin separación de autoridad. La herramienta construye solicitudes solo a partir de los argumentos de modelo declarados, la señal y el owner; los llamadores confiables dentro del proceso pueden fijar los campos de la solicitud directamente. Las variables propiedad del harness usan el canal aparte `dshEnv` de la [decisión de entorno gestionado](../feature/2026-07-10-agent-session-identity-and-log-location.es.md), así que un `env` ordinario no puede reemplazarlas.
2. **`env` se fusiona DESPUÉS del saneamiento de credenciales, así que una entrada explícita del llamador gana incluso con un nombre con forma de credencial.** La decisión posterior de espacio de nombres gestionado gestiona `DSH_*`: las entradas ambientales se eliminan, y el `dshEnv` confiable se fusiona el último, así que una entrada `env` ordinaria nunca puede desplazar un valor gestionado. El orden completo es `scrub(process.env, including DSH_*)` → `ENV_OVERRIDES` → `env` ordinario → `dshEnv`.
3. **`stdin`/`env` son obligatorios-ausentes-OK (opcionales simples) en el spec resuelto, NO obligatorios-pero-anulables como `owner`.** `owner` es obligatorio-pero-anulable porque un owner ausente *en silencio* produce una tarea sin dueño y legible entre sesiones — un riesgo de seguridad que un `undefined` visible previene. `stdin`/`env` no tienen ese peligro: uno ausente significa «sin stdin / sin env adicional», que es el caso seguro y ordinario (toda llamada impulsada por el modelo). Así que siguen siendo opcionales simples, igual que `signal`.

`dsh-bash-local` crea un pipe de stdin solo cuando se aportan bytes; en caso contrario, fd 0 sigue siendo `/dev/null`, conservando el comportamiento previo. Escribe los bytes y cierra el pipe. Un `EPIPE` de un hijo que sale sin leer se ignora porque el código de salida del comando y su salida determinan el resultado.

## Alternativas consideradas

**Saneamiento configurable de secretos ambientales.** Rechazada por especulativa. Los llamadores confiables pueden aportar explícitamente los valores requeridos después del saneamiento sin debilitar la protección ambiental por defecto.

## Consecuencias

Los puentes de hooks pasan las cargas útiles JSON y las variables específicas de hook a través del seam de bash existente, conservando su comportamiento de grupo de procesos, truncado y derrame. El comportamiento orientado al modelo permanece sin cambios, y la herramienta bash sigue siendo la única dueña de la construcción de solicitudes de llamadas del modelo. El vocabulario vive en [la referencia de estructuras de datos de bash](../../../../docs/subsystems/shell.es.md).
