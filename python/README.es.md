# DeepSeek Harness Python SDK

[English](README.md) | Español

Paquetes de Python para conducir DeepSeek Harness como subproceso. El SDK cliente se comunica con el runtime incluido mediante JSON-RPC delimitado por saltos de línea sobre stdio.

## Paquetes

| Directorio | Distribución / módulo | Rol |
|---|---|---|
| [sdk](sdk/README.es.md) | `deepseek-harness-sdk` / `deepseek_harness` | API de turnos de alto nivel y cliente JSON-RPC de bajo nivel |
| [sdk-runtime](sdk-runtime/README.es.md) | `deepseek-harness-runtime-bin` / `deepseek_harness_runtime` | Binarios del runtime incluido y configuración de agent por defecto |

## Comportamiento

El SDK arranca el runtime incluido que corresponde, salvo que quien llama seleccione un canal explícito. El cliente selecciona el canal y aporta la configuración por defecto; el runtime en sí siempre exige una configuración explícita. La [referencia del SDK](sdk/README.es.md) y la [referencia del portador del runtime](sdk-runtime/README.es.md) son dueñas de los contratos completos de selección de runtime y de configuración.

## Flujos de trabajo de contribución

Los [flujos de trabajo de contribución en Python](development.es.md) cubren la compilación de artefactos del runtime, la validación de los paquetes, el desarrollo en modo fuente y la distribución.
