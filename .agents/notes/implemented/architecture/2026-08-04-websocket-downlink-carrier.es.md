# Agent Note: WebSocket como transporte para los downlinks del navegador

Status: implemented

[English](2026-08-04-websocket-downlink-carrier.md) | Español

## Problema

La GUI web del navegador ha usado durante mucho tiempo dos respuestas SSE para `events.mux` y `events.host`. Los navegadores HTTP/1.1 normalmente solo permiten unas seis conexiones concurrentes por origin; que cada página ocupe dos permanentemente hace que las pestañas del mismo origin, los recursos de plugins y los RPC ordinarios compitan por los huecos de conexión, y llegar al límite hace que las peticiones se pongan en cola en lugar de solo ralentizarse. El propio protocolo RPC es independiente del canal: una restricción del transporte físico del navegador no debe filtrarse a la capa de objetos de sesión/runtime.

## Decisión

El transporte real del navegador abre un WebSocket independiente por cada clase de flujo de downlink: `/api/events.mux` envía solo `MuxFrame`, y `/api/events.host` envía solo `HostFrame`. Cada mensaje de texto es un documento JSON `ServerRequest` completo; el cliente sigue validando primero el envelope y después la unión concreta de frames para ese camino, y pasa la forma estrecha `RpcRequest<Frame>` al `ConnectionController` existente. Los flujos conservan ciclos de vida independientes y no ofrecen ninguna garantía de orden entre flujos; que termine cualquiera de los dos sigue haciendo fallar toda la generación de conexión y la reconstruye bajo la política de backoff existente.

El WebSocket transporta solo el downlink host→navegador. Todas las llamadas unarias cliente→host y las operaciones `respond` para peticiones de servidor siguen usando el `POST /api/*` existente; el WebSocket no acepta mensajes de aplicación del cliente. `WebApiClient` por tanto mantiene el `fetch` HTTP para el uplink y el WebSocket para el downlink, mientras que el fixture y `InProcessApiClient(toFetchHandler(api))` siguen implementando la misma abstracción `IApiClient` de dos flujos. El transporte de fetch en proceso conserva la codificación y decodificación SSE para verificar el isomorfismo del protocolo independiente del canal, pero las peticiones GET de red a `/api/events.*` solo responden Upgrade Required y no ofrecen un fallback de compatibilidad del navegador.

## Límites de upgrade y ciclo de vida

`dsh-host-webserver` aporta un punto de registro exacto de rutas de upgrade junto a las rutas ordinarias, despacha los sockets de upgrade de Node solo por pathname, contiene los errores de socket crudo y espera a que las conexiones actualizadas supervivientes se cierren durante el teardown del servidor; no sabe nada de frames del Harness ni de mensajes WebSocket. `dsh-client-connection` es el dueño del handshake WebSocket, de la salida de frames y de la cancelación de flujos, y reutiliza la valla de confianza `/api` de Host/Origin antes del upgrade. Una autoridad no confiable o un Origin entre-orígenes se rechaza antes de que arranque `ctx.apiProxy.events.*`.

Un abort del navegador o un cierre de socket cancela el flujo de host correspondiente; el teardown de plugins también espera la limpieza de ese iterador de fuente. Si un flujo de host lanza una excepción a mitad de camino, el transporte envía un frame `stream/error` existente y después cierra el socket; el cliente trata ese frame como pérdida de conexión en lugar de entregarlo a un sumidero de aplicación. Cada WebSocket informa de open de forma independiente, y el handshake de readiness existente sigue esperando a que mux y host estén ambos abiertos y a que la llamada HTTP `host.describe` haya tenido éxito antes de publicar connected.

## Verificación

Los tests de contrato del webserver fijan el despacho por pathname de upgrade, el rechazo de registros duplicados, la disposición y el teardown; los tests de red real de conexión fijan la comprobación de confianza de cada WebSocket, el open, el envelope del schema, el orden de frames, el error de flujo y la cancelación por cierre; los tests del cliente también demuestran que los downlinks crean URLs `ws:`/`wss:` mientras que las llamadas unarias y `respond` siguen usando el `fetch` HTTP. El replay de navegador sin clave ensamblado sigue cubriendo Chromium, un host real, el uplink HTTP y toda la cadena de downlink WebSocket.

## Alternativas consideradas

**Multiplexar mux y host sobre un único WebSocket.** Esto añadiría una etiqueta de canal, una cola de multiplexación y una política de backpressure de conexión única, y cambiaría la semántica de readiness existente de dos flujos. Dos WebSockets ya evitan el límite de seis conexiones de HTTP/1.1 manteniendo este cambio en la capa de transporte físico.

**Mover también las llamadas unarias y respond a un WebSocket full-duplex.** Esto reescribiría el comportamiento de timeout, cancelación, estado HTTP, valla de confianza y correlación de peticiones sin añadir ningún beneficio para el problema actual de huecos de conexión del downlink. El uplink HTTP es una frontera conservada explícitamente.

**Conservar un fallback SSE de red.** Dos transportes dejarían que el camino del navegador en producción se bifurcara en silencio por diferencias de proxy o handshake y dejarían el problema del límite de conexiones en una rama soportada. Durante la prepublicación solo se distribuye el downlink WebSocket; el comportamiento de reconexión existente y el estado de conexión exponen los fallos explícitamente.

**Confiar en HTTP/2 para una mayor concurrencia de conexiones.** El servidor de desarrollo integrado usa Node HTTP/1.1 en texto plano, y el proxy frontal de un despliegue no es un invariante del producto. El downlink físico usa directamente una primitiva del navegador fuera de ese grupo de conexiones.

## Consecuencias

Cada página web sigue teniendo dos conexiones de downlink de larga duración, pero ya no consumen la cuota de seis conexiones HTTP/1.1 del navegador. El runtime sigue consumiendo los dos flujos originales y conserva toda la semántica de reconexión, reparación de flujos y desorden entre flujos. El coste es una superficie más de registro de upgrade en el webserver, una dependencia de implementación WebSocket en la mitad host del paquete de conexión, y el mantenimiento por separado de los códecs físicos del WebSocket del navegador y del SSE en proceso. Comparten los mismos schemas `ServerRequest`/frame y la semántica de `IApiClient`, evitando un segundo protocolo de aplicación.
