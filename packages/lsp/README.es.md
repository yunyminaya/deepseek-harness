# lsp/ - familia de capacidad LSP

[English](README.md) | Español

El seam de capacidad de servidor de lenguaje: una Service Definition de LSP, un provider stdio genérico y la herramienta `lsp` orientada al modelo. Todos paquetes de **producto**.

| Paquete | Rol | Clave de ctx |
|---|---|---|
| `lsp/` | Service Definition (registro de providers por id con marca + mapeo de extensiones, selección por consulta, vocabulario, `LspError`) | `ctx.lsp` |
| `lsp-stdio/` | Backend stdio genérico de varios servidores sobre `ctx.fs` y `ctx.subprocess` (JSON-RPC, consultas de apertura transitoria) | (registra providers en `ctx.lsp`) |
| `tool-lsp/` | Herramienta `lsp` orientada al modelo (cuatro operaciones, coordenadas de cursor UTF-16 de base uno) | (registra en `ctx.tools`) |

La Service Definition vive en `lsp/lsp/`. El seam expone exactamente cuatro operaciones semánticas — `goToDefinition`, `findReferences`, `goToImplementation`, `hover` — y ninguna salida de escape JSON-RPC genérica, de modo que un intercambio de provider no cambia cómo pide el modelo la navegación y ningún payload de protocolo ni mutación sin revisar llega al contrato del modelo. Los providers registran **capacidades**, no herramientas; `tool-lsp` es el único dueño del nombre orientado al modelo, del schema, de la guía de prompt y de la presentación.

Ver el [Agent Note del seam de capacidad LSP](../../.agents/notes/implemented/architecture/2026-07-15-lsp-capability-seam.es.md) para el razonamiento de diseño, incluido por qué los documentos se abren transitoriamente por consulta, por qué el host stdio consume el mundo de ejecución compartido de filesystem/subprocess y por qué la propiedad de extensiones es exclusiva dentro de un runtime.

La referencia del subsistema — operaciones, coordenadas, peticiones/resultados, `LspError` — está en [docs/subsystems/lsp.md](../../docs/subsystems/lsp.es.md); el razonamiento de diseño, en el [Agent Note del seam de capacidad LSP](../../.agents/notes/implemented/architecture/2026-07-15-lsp-capability-seam.es.md).
