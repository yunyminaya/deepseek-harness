# spill/ — familia de capacidades de spill de salida de herramientas

[English](README.md) | Español

Esta familia persiste la salida sobredimensionada de las herramientas y reemplaza el resultado en línea con una vista previa acotada y un localizador de recuperación.

| Paquete | Rol | Clave de ctx |
|---|---|---|
| [`spill/`](spill/README.es.md) | Define el almacenamiento de spill | `ctx.spillStore` |
| [`spill-local/`](spill-local/README.es.md) | Almacena texto con spill en archivos locales con ámbito de sesión | se registra en `ctx.spillStore` |
| [`spill-policy/`](spill-policy/README.es.md) | Aplica la política de spill posterior a la ejecución | escucha en `ctx.tools` |

Consulta la [decisión de spill de salida de herramientas](../../.agents/notes/implemented/architecture/2026-07-08-tool-output-spill-files.es.md) para el límite entre almacenamiento, retención y manejo de salida propiedad de la herramienta.

La referencia del subsistema — `SaveTextSpill`, propietarios/fuentes, el localizador con marca — está en [docs/subsystems/spill.md](../../docs/subsystems/spill.es.md); la justificación, en la [Agent Note de spill de salida de herramientas](../../.agents/notes/implemented/architecture/2026-07-08-tool-output-spill-files.es.md).
