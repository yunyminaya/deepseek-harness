# `@deepseek-ai/dsh-host-frontend-static`

[English](README.md) | Español

Servidor de dist de SPA para el shell web: un plugin de función (config `{distIndex}`) que ocupa el único asiento de reserva del [webserver](../webserver/README.es.md) y sirve el directorio de frontend compilado con puntos de entrada de índice explícitos. Mientras `distIndex` sea legible, la raíz de dist y la ruta de índice configurada renderizan `index.html` con HTTP 200; el resto de archivos existentes se sirven directamente. Un destino ausente o que no sea un archivo dentro de la raíz de dist, incluido un índice configurado inexistente, devuelve un 404 vacío; el traversal fuera de la raíz de dist devuelve 403, las extensiones desconocidas se sirven como `application/octet-stream` y las peticiones que no son GET/HEAD sin una ruta con nombre que coincida devuelven 405. Toda respuesta de índice correcta se renderiza a través de `renderIndex` del webserver — primero las filas de inyección estructuradas, luego los taps de índice crudos —, que es como el manifest de arranque llega a la página. `distIndex` es un hecho de ensamblaje de la aplicación que compone: [`dsh-web-app`](../../bundle/web-app/README.es.md) lo resuelve a través de los exports del paquete de frontend y monta este plugin; un despliegue nunca lo fija en código.

El asiento de reserva es de propietario único (una segunda reclamación lanza una excepción) y tiene ámbito de effect: hacer dispose del fiber del plugin libera el asiento y, a partir de entonces, el webserver sin reclamar responde 404.

## Experiencia del modelo

Ninguna, ya que el paquete sirve recursos del navegador; nada de esto llega a una solicitud de modelo.

#### Efecto en la caché KV

Ninguno; este paquete ni ensambla ni envía una solicitud de provider.

## Limitaciones conocidas y trabajo diferido

- **La tabla MIME inicial es mínima** — cubre el conjunto de recursos emitidos por Vite más el manifest PWA incluido; otras extensiones recurren a `application/octet-stream` hasta que una clase de recursos se incluya de verdad.
- **El enrutamiento por pathname es explícito** — el cliente actual entra por la raíz o por la ruta de índice configurada y no tiene rutas de pathname de History API. Añadir una exige una regla de servidor explícita y cobertura de composición real, no un fallback amplio para cada fallo.
