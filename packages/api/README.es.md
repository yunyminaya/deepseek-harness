# api/ — Capas de API remota

[English](README.md) | Español

La pila Remote orientada a la aplicación. `remotes` es dueño de la política del BFF y de la API de negocio seleccionada, mientras que `gateway` implementa los endpoints RPC unarios de Typert compartidos por los entornos Host y Client.

| Paquete | Rol | Clave de ctx |
|---|---|---|
| [`remotes/`](remotes/README.es.md) | Política de búsqueda de Agent/Sesión del Host y ensamblaje de contribuciones Remote del Client | sin servicio; configura `ctx.typert` y consume `ctx.remote` |
| [`gateway/`](gateway/README.es.md) | Despachador Typert del Host y endpoint Remote del Client | `ctx.typertGateway` / `ctx.remote` |

La dirección de dependencias en runtime es `remotes → gateway → connection → webserver`: el BFF consume el contrato compartido `TypertClientRemote`, Gateway delega el transporte en Connection y Connection se monta en el servidor HTTP. La inyección de servicios de Cordis y los metadatos de módulos del Client conservan este orden sin importar el Gateway concreto desde la entrada Remotes Client.

## Limitaciones conocidas y trabajo diferido

- Connection y WebServer permanecen en [`client/connection`](../client/connection/README.es.md) y [`host/webserver`](../host/webserver/README.es.md); un movimiento posterior solo de paquetes puede colocarlos en `api/connection` y `api/webserver` sin cambiar sus contratos de servicio.
- El API Proxy heredado permanece en [`host/apiproxy`](../host/apiproxy/README.es.md) como respaldo de los métodos aún no migrados a Remote. Consume el resolver del Host propiedad de `api-remotes` para que los métodos migrados y los heredados conserven una única política de identidad de Agent/Sesión.
