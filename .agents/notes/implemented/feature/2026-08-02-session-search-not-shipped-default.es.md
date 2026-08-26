# Agent Note: Las herramientas de búsqueda de sesiones no son un valor por defecto publicado

Status: implemented

[English](2026-08-02-session-search-not-shipped-default.md) | [中文](2026-08-02-session-search-not-shipped-default.zh.md) | Español

## Problema

La [decisión de rosters de herramientas publicados](2026-07-31-even-out-shipped-tool-rosters.es.md) convirtió `tool-session-query` en una fila por defecto del [`cordis.patch.yml`](../../../../packages/bundle/base/cordis.patch.yml) compartido, así que las superficies TUI y Web publicadas pusieron las cinco herramientas de búsqueda de sesiones (`session_search`, `session_event_search`, `session_trace`, `session_event_trace`, `session_event_read`) delante del modelo. Eso contradecía la [decisión de herramientas de consulta de sesión visibles para el modelo](2026-07-24-model-facing-session-query-tools.es.md), cuya postura opt-in el README del paquete registraba como «las composiciones de host publicadas no lo montan por defecto». El valor por defecto también publicaba una sección de prompt que enseñaba un flujo de trabajo de búsqueda de trabajos anteriores que ningún usuario había pedido.

## Decisión

Las superficies TUI, Web y headless publicadas no montan `@deepseek-ai/dsh-tool-session-query`, y ningún preset de agent publicado lo lleva. El Consumer sigue siendo opt-in exactamente como describe la nota de herramientas de consulta de sesión visibles para el modelo: el [`session-query.cordis.yml`](../../../../examples/acp-agent/session-query.cordis.yml) del ejemplo ACP y su contrapartida de instantánea siguen siendo la referencia montada, y una composición personalizada puede montar el paquete con las políticas de timeout y spill.

El servicio `ctx.sessionQuery` en sí sigue montado. `session-query-sqlite` sigue siendo una fila de la base — el `session-reference` de la TUI lo consume para `/resume` — con su índice de texto completo desactivado por defecto (`openAt: never`; la [decisión de opt-in de búsqueda de contenido](../architecture/2026-08-13-session-content-search-opt-in.es.md)), y el overlay Web conserva sus valores en memoria para los despliegues que habilitan la búsqueda de contenido. Solo se elimina el Consumer visible para el modelo.

## Alternativas consideradas

- **Eliminar también el índice de `session-query-sqlite`** — rechazada porque `/resume` y la caja de búsqueda de contenido Web consumen `ctx.sessionQuery` directamente; son funcionalidades del host, no herramientas del modelo, y eliminar el provider las rompería.
- **Mantener la fila pero desactivarla en cada overlay** — rechazada porque una fila de la base desactivada sigue publicando la dependencia e invita a una re-activación de una línea; la postura opt-in registrada quiere el Consumer ausente de las superficies publicadas, con el ejemplo ACP como referencia de montaje.
- **Montarla solo en la TUI** — rechazada porque la base compartida es un único conjunto de filas para cada superficie; un montaje específico de superficie reintroduciría la división de rosters que la decisión de rosters publicados eliminó.

## Consecuencias

Ambas superficies vuelven a las mismas veinte herramientas incondicionales (más `glob`/`grep` bajo ripgrep), y los cinco schemas de búsqueda de sesiones y su sección de prompt abandonan la solicitud por defecto. Las pruebas de composición publicada en ambas superficies fijan el catálogo más pequeño, así que volver a añadir la búsqueda de sesiones como valor por defecto toca las mismas pruebas. Los usuarios que quieran búsqueda de sesiones montan el Consumer desde un overlay personal o el ejemplo ACP, añadiendo la dependencia donde lo hagan.
