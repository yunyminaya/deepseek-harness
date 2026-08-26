# boot/ — pegamento de arranque compartido de los app bins
[English](README.md) | Español

La biblioteca de arranque neutra respecto al canal que comparten `apps/cli` y los bins de demo de [`examples/`](../examples/README.es.md).

| Paquete | Rol | clave ctx |
|---|---|---|
| `app-boot/` | Pegamento de arranque compartido para los app bins: carga de `.env`, guardas de Loader que fallan en voz alta, resolución de configuración consciente de instantáneas, la secuencia de arranque que asienta el árbol | (biblioteca para los bins) |
| `cmdline/` | Entrega de la línea de comandos del lanzador a la app y análisis de arranque propiedad de la app | `cmdlineArgs`, `appExit` |

La secuencia de arranque y el contrato de configuración personal se documentan en [`app-boot/README.md`](app-boot/README.es.md); las líneas de comandos propiedad de la app se documentan en [`cmdline/README.md`](cmdline/README.es.md).
