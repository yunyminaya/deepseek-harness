# hooks/ — puentes de hook + protocolo compartido

[English](README.md) | Español

El subsistema hooks permite a los usuarios extender el agent en puntos del ciclo de vida de la misma manera que hacen Claude Code y Codex — apuntando un plugin puente a un `hooks.json` existente (o a los ajustes) para que esos hooks de shell externos se ejecuten fielmente. La superficie de extensión canónica en sí es la de los puntos de intercepción tipados del harness ([el Agent Note de puntos de intercepción](../../.agents/notes/implemented/feature/2026-06-30-interception-extension-points.md)); un «hook nativo» es solo un plugin de Cordis ordinario sobre esos puntos de extensión. Estos paquetes son los **puentes** que traducen el protocolo de hooks de shell externo sobre esa misma superficie, más la librería de protocolo de cable compartida sobre la que se construyen.

| Paquete | Rol | Forma |
|---|---|---|
| [`hook-protocol/`](hook-protocol/README.es.md) | Librería compartida del protocolo de hooks de shell | librería |
| [`hooks-claude-code/`](hooks-claude-code/README.es.md) | Puente de hooks de Claude Code | plugin |
| [`hooks-codex/`](hooks-codex/README.es.md) | Puente de hooks de Codex | plugin |

La librería compartida es dueña del comportamiento común del protocolo; cada puente es dueño de su mapeo de eventos específico del dialecto. Los README hijos documentan esos contratos.
