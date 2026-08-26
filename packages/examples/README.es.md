# examples/ — bundles de demo listos para ejecutar

[English](README.md) | Español

Bundles de plugins precompuestos que una hoja (leaf) fina de `cordis.yml` carga en lugar de ensamblar el spine y un punto de entrada a mano. Son paquetes **demo / referencia** — el sufijo npm `-demo` marca cada uno como superficie no de producto, legible directamente del nombre del paquete. Las hojas ejecutables bajo el [`examples/`](../../examples/AGENTS.es.md) de la raíz del repo y el [runtime del SDK de Python](../../python/sdk-runtime/README.es.md) son los consumers; cada uno es solo sus backends intercambiables más una entrada de bundle.

| Paquete | Nombre npm | Rol |
|---|---|---|
| [`agent-spine-demo/`](agent-spine-demo/README.es.md) | `@deepseek-ai/dsh-agent-spine-demo` | Bundle reutilizable de agent-spine |
| [`acp-demo/`](acp-demo/README.es.md) | `@deepseek-ai/dsh-acp-demo` | Bundle de aplicación de automatización ACP |
| [`jsonrpc-demo/`](jsonrpc-demo/README.es.md) | `@deepseek-ai/dsh-sdk-jsonrpc-demo` | Runtime JSON-RPC con configuración externa |

`agent-spine-demo` es el bundle compartido; `acp-demo` añade su punto de entrada de automatización, mientras que `jsonrpc-demo` arranca un árbol de plugins propiedad del despliegue. La ejecución única de producto pertenece a `dsh --profile headless`; ningún paquete de este directorio la ofrece.

Estos paquetes no son API de producto. Los seams y los puntos de entrada de producto permanecen en sus grupos propietarios; los bundles de demo eligen composiciones concretas.

No confundas este grupo con el [`examples/`](../../examples/AGENTS.es.md) de la raíz del repo: ese directorio contiene las **hojas** `cordis.yml` ejecutables; este grupo contiene los **bundles** que esas hojas cargan.
