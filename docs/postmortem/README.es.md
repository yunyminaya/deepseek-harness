# Post-mortems

[English](README.md) | Español

Relatos de incidentes: un bug llegó a un lugar al que no debía (un usuario real, un PR fusionado, un release), y la parte interesante es *por qué nuestro proceso lo dejó pasar*, no solo la corrección de una línea.

Un post-mortem NO es un [Agent Note](../../.agents/notes/README.es.md) (que registra una decisión de diseño deliberada y sus alternativas rechazadas, o propone trabajo futuro). Es un registro retrospectivo de un fallo: qué se rompió, el mecanismo, por qué todas las redes de seguridad lo dejaron pasar y las protecciones concretas añadidas para que la misma clase de bug falle ruidosamente la próxima vez.

Escribe uno cuando un bug sea **sutil** (el mecanismo no es evidente y un ingeniero cuidadoso lo re-derivaría por el camino difícil), **sistémico** (la razón por la que se escapó es un hueco en pruebas/herramientas/convenciones, no un error tipográfico puntual) y **costoso de redescubrir** (costó tiempo real de depuración, y lo volvería a costar). Enlaza las protecciones (pruebas, reglas de AGENTS.md, ADRs) que motivó el post-mortem.

Todo post-mortem comienza con un **Resumen ejecutivo**: un párrafo breve que un lector ocupado puede asimilar en treinta segundos — qué se rompió, la causa raíz en términos llanos, por qué se escapó y la lección duradera — antes de las secciones detalladas de Resumen / Cronología / Causa raíz / Protecciones que le siguen.

| # | Título |
|---|---|
| [0001](0001-acp-default-export-drops-inject.es.md) | El servidor de ACP fallaba al conectar: `export default` descartó el `inject` del plugin |
| [0002](0002-js-expression-disabled-filesystem-tools.es.md) | Las herramientas de instantánea del sistema de archivos quedaron permanentemente deshabilitadas por un objeto `!!js` literal |
| [0003](0003-web-agent-gui-feedback-loop.es.md) | El Web agent validó un servidor de reemplazo en lugar de la GUI que alojaba su sesión |
| [0004](0004-landlock-partial-notice-misclassified-child-failures.es.md) | El aviso de aplicación parcial de Landlock clasificó mal los fallos de los hijos |
