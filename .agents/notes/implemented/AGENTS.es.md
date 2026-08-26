# AGENTS.md — Implemented Agent Notes

[English](AGENTS.md) | Español

Estas Agent Notes describen decisiones implementadas. Sigue las [instrucciones raíz](../../../AGENTS.md), el [estándar de documentación](../../../docs/AGENTS.md) y el [formato de Agent Note](../README.md#the-file-format); `verify-agent-note-format` controla la estructura específica del ciclo de vida.

## Mantener una Agent Note implementada actualizada con lo que realmente se envió

Mantén las rutas, símbolos, predeterminados y mecanismos actualizados en el mismo cambio que los altera. Reescribe los hechos obsoletos en su lugar; no anexes el historial de cambios.

Cuando una nota enviada es poco probable que guíe el trabajo futuro, archiva su tríplete completo a través de [`dsh-archive-agent-notes`](../../skills/dsh-archive-agent-notes/SKILL.md) en lugar de continuar manteniéndola.

### Esto no es una licencia para reescribir la *decisión*

Actualiza la realización fáctica en su lugar. Una reversión de la decisión o su justificación requiere una nueva Agent Note y un enlace cruzado; una nota antigua completamente suplantada solo puede eliminarse a través de la regla de consolidación en las [reglas de Agent Notes](../README.md).