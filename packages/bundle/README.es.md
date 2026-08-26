# bundle/ — bundles de plugins de profile
[English](README.md) | Español

Bundles de profile: paquetes npm cuyo manifest declara `"dsh": { "bundle": { "patch": "./cordis.patch.yml" } }`, lo que los convierte en capas de patches instalables para las composiciones `dsh --profile` ([contrato de profile](../boot/app-boot/README.es.md#profiles)). La sustancia de un bundle es su lista de patches; algunos también distribuyen plugins de pegamento de runtime que su patch monta.

La declaración del manifest, no este directorio, define la identidad de Bundle. Los paquetes de dominio pueden llevar su propia capa de Profile opcional; los [paquetes de subagente Codex y Claude Code](../subagent/README.es.md) son ejemplos directamente instalables.

| Paquete | Rol | clave ctx |
|---|---|---|
| [`base/`](base/README.es.md) | El núcleo dsh compartido que todo profile aplica primero | — (solo patches) |
| [`web-app/`](web-app/README.es.md) | Superficie de navegador: capa de patches web + plugin de pegamento de runtime | monta filas |
| [`headless/`](headless/README.es.md) | Modo de tarea única directo sobre base, sin capa de Host ni de Web | monta `headless-runner` |

Los bundles dentro de la caja se resuelven desde la instalación de dsh; los bundles fuera del árbol se instalan en un profile mediante `dsh plugin --profile <name> add <package>`.
