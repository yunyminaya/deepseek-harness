# host/ — la mitad host de la GUI web

[English](README.md) | Español

El lado host de la GUI web de dsh: el gateway de API que comparte toda forma de cliente, y el servidor HTTP plano sobre el que viaja. El lado del navegador vive en [`client/`](../client/README.es.md); la aplicación compuesta es [`apps/cli`](../../apps/cli/README.es.md) arrancando el bundle [`dsh-base`](../bundle/base/cordis.patch.yml) que sirve [`apps/web`](../../apps/web/). Todos son paquetes **producto**.

| Paquete | Rol | Clave ctx |
|---|---|---|
| [`apiproxy/`](apiproxy/README.es.md) | Gateway de API de host compartido y contrato de cable | `ctx.apiProxy` |
| [`webserver/`](webserver/README.es.md) | Transporte de rutas HTTP | `ctx.webServer` |
| [`frontend-static/`](frontend-static/README.es.md) | Servidor de dist SPA en el asiento de respaldo del webserver | consume `ctx.webServer` |
| [`directory-picker/`](directory-picker/README.es.md) | Seam de elección de directorio de workspace | `ctx.directoryPicker` |
| [`directory-picker-native/`](directory-picker-native/README.es.md) | Backend nativo de elección de directorio e interacción con el navegador | registra `ctx.directoryPicker` |
| [`directory-picker-browse/`](directory-picker-browse/README.es.md) | Backend de navegador de directorios en la app e interacción | registra `ctx.directoryPicker` |
| [`directory-picker-auto/`](directory-picker-auto/README.es.md) | Composición de picker adaptable al host | monta un backend |
| [`plugin-inventory/`](plugin-inventory/README.es.md) | Proyección de solo lectura de las entradas actuales del Loader | Remote `pluginInventory/list` |

`apiproxy` permanece independiente del transporte; [`client/connection`](../client/connection/README.es.md) suministra el carrier de navegador/HTTP. Las implementaciones de picker se reemplazan unas a otras tras el seam compartido.

El subsistema referencia: [web-server.md](../../docs/subsystems/web-server.es.md) y [workspace.md](../../docs/subsystems/workspace.es.md) (el seam del picker).
