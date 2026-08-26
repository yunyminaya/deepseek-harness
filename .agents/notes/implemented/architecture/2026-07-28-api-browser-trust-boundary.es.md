# Agent Note: Un único límite de confianza de navegador a nivel de carrier para todas las rutas `/api`

Status: implemented

[English](2026-07-28-api-browser-trust-boundary.md) | Español

## Problema

El host de la GUI web sirve `/api` sobre HTTP plano (por defecto `127.0.0.1:3080`, con `--host 0.0.0.0` soportado), y la superficie incluye métodos de nivel de ejecución remota de código — `session.prompt` impulsa un agent que ejecuta bash. Un navegador convierte al operador en un deputy confundido contra una API local así de dos maneras clásicas: una página maliciosa dispara un POST cross-site «simple» (`text/plain` — enviado sin preflight CORS) cuyos efectos secundarios se ejecutan aunque la respuesta permanezca ilegible, y un origen de DNS rebinding habla con el socket como si fuera same-origin, haciendo CORS inaplicable por completo, con solo el header `Host` delatando el dominio del atacante. Antes de esta decisión la única verificación de confianza de navegador del sistema (`isTrustedNativeDialogRequest`: socket loopback + same-origin + Host loopback) protegía exactamente una ruta cosmética — `host.pickDirectory`, cuyo diálogo nativo se abre en la pantalla del host — mientras todo método importante estaba desprotegido. Proteger por RPC tampoco podía sobrevivir al navegador de directorios dentro de la app, cuyo propósito mismo es servir clientes legítimamente remotos que una regla de loopback rechazaría.

## Decisión

Imponer la confianza de navegador una sola vez, en el carrier, para todo el prefijo `/api` — dos mitades:

- **Valla de tipo de medio (dsh-host-apiproxy)**: todo POST de `/api` debe declarar `application/json`, si no 415 antes de parsear. Las peticiones cross-site «simples» dejan así de existir: cualquier intento cross-site queda forzado a un preflight CORS que este servidor nunca responde.
- **Valla de autoridad (dsh-client-connection, `src/api-request-trust.ts`)**: toda petición debe presentar un `Host` que sea loopback o coincida con una entrada de `trustedHosts` (exacta en `host:port`, cualquier puerto en entradas sin puerto, normalizada según WHATWG; defensa contra rebinding). Deliberadamente sin atajo para peticiones sin marcar: sobre HTTP plano un navegador no adjunta `Origin` ni Fetch-Metadata a las lecturas (EventSource, imágenes, navegaciones — esos headers van solo a destinos de confianza), así que una petición sin marcar puede ser una lectura de navegador con rebinding cuya respuesta la página puede leer, y Host es el único header que el rebinding no puede forjar; los clientes que no son navegador pasan por loopback, los literales de IP LAN derivados o una autoridad declarada. Un `Origin` adjunto debe igualar la autoridad del Host; `sec-fetch-site: cross-site` se rechaza de plano. Una entrada de `trustedHosts` que no sea una autoridad canónica desnuda hace fallar la carga del plugin — el parseo WHATWG autorizaría silenciosamente el hostname dentro de un error tipográfico o ampliaría una concesión de puerto exacto. `host.pickDirectory` pierde su guarda hecha a medida y monta la misma valla.

Dos límites quedan deliberadamente fuera de alcance: la alcanzabilidad es política del binding del webserver (`host: 127.0.0.1 | 0.0.0.0`), y la autenticación para despliegues genuinamente remotos es trabajo diferido registrado en el README de la conexión — la valla es una defensa de deputy confundido, no una capa de auth. La comprobación de socket loopback de la antigua guarda se descartó en lugar de generalizarse: con el binding expresando alcanzabilidad y `trustedHosts` nombrando autoridades remotas, la dirección de socket no añade nada que una valla de headers no cubra ya.

## Alternativas consideradas

- **Guarda por RPC (status quo extendido).** Rechazada: la lista de guardas persigue a la lista de métodos para siempre, los métodos de mayor valor ya estaban sin proteger, y una regla de loopback en los RPC de browse rompería los despliegues remotos para los que existen.
- **Headers CORS + omisión de credenciales.** Rechazada: no queremos lecturas cross-origin en absoluto, así que responder preflights solo ensancha la superficie; rechazarlos es estrictamente más fuerte y más simple.
- **Tokens de auth ahora.** Rechazada para este cambio: la acuñación/almacenamiento/rotación de tokens es superficie real de producto; la valla cierra los agujeros de deputy de navegador hoy sin predecir el diseño de auth.

## Consecuencias

- Todo método futuro de `/api` queda cubierto por construcción; no queda ninguna decisión de confianza por ruta que olvidar.
- Los despliegues no-loopback deben tener confiadas sus autoridades servidoras o las peticiones se rechazan. El CLI dsh mantiene funcionando su URL LAN anunciada `--host 0.0.0.0` derivando los literales de IP LAN de la máquina a la fila de conexión (entradas sin puerto — un Host de IP literal no puede ser un nombre con rebinding, y el puerto enlazado puede ser asignado por el OS) y ofrece `dsh web --trusted-host` para autoridades nombradas; las composiciones que el CLI no arranca declaran `trustedHosts` ellas mismas. La automatización no-navegador monta la misma valla: pasa por loopback, una IP LAN derivada o una autoridad declarada; un alias DNS no declarado se rechaza.
- Los clientes deben etiquetar los cuerpos de POST como `application/json` (los nuestros siempre lo hicieron; los tests de fetch crudo ganaron el header).
- El supuesto de red de confianza de un despliegue no autenticado en `0.0.0.0` queda ahora documentado en lugar de implícito.
