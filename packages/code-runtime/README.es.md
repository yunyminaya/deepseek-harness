# code-runtime/ — familia de capacidades de ejecución de código

[English](README.md) | Español

El seam de capacidad de ejecución de código (ver [capability seams](../../.agents/notes/implemented/architecture/2026-06-13-capability-seams.es.md)): una Service Definition de runtime para ejecutar un programa escrito por el modelo contra bindings asíncronos proporcionados por el host, capturando lo que imprimió y devolvió; providers reemplazables; y el Consumer de [Code Mode](../core/tools/README.es.md) del registro de herramientas (`tools: { mode: code }` — la herramienta `run_code` y el SDK generado en el `language` del runtime cargado). El diseño está en la [Agent Note de Code Mode](../../.agents/notes/implemented/feature/2026-06-15-code-mode.es.md). Paquetes **Product**.

| Paquete | Rol | clave ctx |
|---|---|---|
| [`code-runtime/`](code-runtime/README.es.md) | Service Definition y vocabulario compartido | `ctx.codeRuntime` |
| [`code-runtime-worker/`](code-runtime-worker-thread/README.es.md) | Backend de hilo de trabajo | registra `ctx.codeRuntime` |

Los providers registran el servicio sin cambiar su Consumer. Los README hijos son dueños de los detalles de lenguaje, aislamiento y presupuesto de ejecución.

La referencia del subsistema — solicitudes/resultados de ejecución, namespaces de bindings, la taxonomía de fallos — es [docs/subsystems/code-runtime.md](../../docs/subsystems/code-runtime.es.md).
