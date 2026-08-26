# compaction/ — familia de capacidades de compactación

[English](README.md) | Español

Una familia de capacidades de compactación (ver [capability seams](../../.agents/notes/implemented/architecture/2026-06-13-capability-seams.es.md)): una Service Definition, un provider de resumen, un compañero de poda de resultados de herramienta sin modelo y un Consumer de comando humano. Todos paquetes **product**.

| Paquete | Rol | clave ctx |
|---|---|---|
| [`compaction/`](compaction/README.es.md) | Seam de compactación y vocabulario de eventos | `ctx.compaction` |
| [`compaction-basic/`](compaction-basic/README.es.md) | Backend de presión de tokens y resumen | registra `ctx.compaction` |
| [`compaction-tool-result-pruner/`](compaction-tool-result-pruner/README.es.md) | Poda opcional de resultados de herramienta sin modelo | `ctx.toolResultPruner` |
| [`command-compact/`](command-compact/README.es.md) | Comando humano de compactación | se registra en `ctx.commands` |

El backend, el pruner opcional y el comando humano se componen a través del seam; la medición de tokens sigue siendo un servicio separado de la familia LLM. La [Agent Note de capability-seam de compactación](../../.agents/notes/implemented/feature/2026-06-18-compaction-capability-seam.es.md) es dueña de la justificación de las dependencias.

La referencia del subsistema — los eventos `compaction/*`, `CompactionResult`, el servicio, los resultados de poda — es [docs/subsystems/compaction.md](../../docs/subsystems/compaction.es.md); la dependencia deliberada del seam de `dsh-session`/`dsh-llm` está registrada en la [Agent Note de capability-seam de compactación](../../.agents/notes/implemented/feature/2026-06-18-compaction-capability-seam.es.md).
