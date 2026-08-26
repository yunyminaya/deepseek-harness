# credentials/ — credenciales y autorización

[English](README.md) | Español

La familia de capacidades de credenciales separa la resolución de referencias de su provider, y separa ambos de la obtención de una credencial que hay que pedir:

| Paquete | Rol | Clave de ctx |
|---|---|---|
| [`credentials/`](credentials/README.es.md) | Seam de referencia de credenciales y de registro de credenciales | `ctx.credentials` |
| [`credentials-local/`](credentials-local/README.es.md) | Provider de entorno y de archivos locales | registra `ctx.credentials` |
| [`authorization/`](authorization/README.es.md) | Flujos propiedad de plugins que obtienen una credencial preguntando a un humano | `ctx.authorization` |

La configuración lleva referencias, no valores secretos. Los Consumers resuelven esas referencias en el límite de su operación; los README hijos poseen la semántica de mutación, precedencia y almacenamiento. Un flujo de autorización escribe un registro de credenciales y ese registro es su clave, de modo que los dos seams se encuentran en el registro y en ningún otro sitio.

La referencia del subsistema — `CredentialRef`, resolución por operación, `CredentialInfo` seguro para la UI, capas de provider — está en [docs/subsystems/credentials.md](../../docs/subsystems/credentials.es.md).
