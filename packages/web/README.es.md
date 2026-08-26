# web/ — familia de capacidades web

[English](README.md) | Español

Esta familia proporciona operaciones de búsqueda web y fetch neutrales respecto al provider, además de las herramientas orientadas al modelo que las consumen.

| Paquete | Rol | Clave de ctx |
|---|---|---|
| [`web/`](web/README.md) | Define el registro, la selección y los errores compartidos de los providers web | `ctx.web` |
| [`web-search-exa/`](web-search-exa/README.md) | Proporciona búsqueda web mediante Exa | se registra en `ctx.web` |
| [`web-search-perplexity/`](web-search-perplexity/README.md) | Proporciona búsqueda web mediante Perplexity | se registra en `ctx.web` |
| [`web-search-deepseek/`](web-search-deepseek/README.md) | Proporciona la búsqueda web nativa de DeepSeek | se registra en `ctx.web` |
| [`web-fetch-http/`](web-fetch-http/README.md) | Obtiene recursos HTTP y HTTPS públicos | se registra en `ctx.web` |
| [`tool-web/`](tool-web/README.md) | Expone la búsqueda web y el fetch al modelo | se registra en `ctx.tools` |

La [decisión de capacidad web](../../.agents/notes/implemented/architecture/2026-06-24-web-capability-seam.md) registra por qué la búsqueda y el fetch comparten un único servicio de selección de providers.

La referencia de subsistema — solicitudes y resultados de búsqueda/fetch, disponibilidad, `WebError` — es [docs/subsystems/web.md](../../docs/subsystems/web.es.md); el fundamento (incluida la protección SSRF diferida) está en la [Agent Note del seam de capacidad web](../../.agents/notes/implemented/architecture/2026-06-24-web-capability-seam.md).
