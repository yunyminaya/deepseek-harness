# `@deepseek-ai/dsh-web-app`
[English](README.md) | Español

El bundle de superficie de navegador de dsh. [`cordis.patch.yml`](cordis.patch.yml) se aplica directamente sobre [`dsh-base`](../base/README.es.md): fija la persona de coding, inserta las filas de host Web (webserver, API gateway, workspace, caché de proyección, storage) y la lista de plugins de navegador, la cadena de recarga de plugins de client siempre activa ([`dsh-client-hmr`](../../client/hmr/README.es.md), inactiva hasta que un watcher de reconstrucción reescribe los bundles de client), y monta el plugin de pegamento `web-runtime` de este paquete (config `{openBrowser, printUrl, surfaceContext, trustedHosts}`). Ese plugin resuelve el dist de frontend compilado a través de los exports de `@deepseek-ai/dsh-web-frontend`, muestrea una vez la confianza LAN dependiente del bind, la aporta como `webRuntime` a la valla de confianza del navegador y a la lista de client, monta el dueño de respaldo [`frontend-static`](../../host/frontend-static/README.es.md) y registra las secciones de prompt harness-source y web-surface más la variable de runtime `DSH_WEB_URL` visible en bash cuando `surfaceContext` es true. Después de que su árbol de Loader se asienta, imprime la línea de URL `dsh web:` cuando `printUrl` es true y abre la URL canónica del host en el navegador predeterminado cuando `openBrowser` es true y los `SSH_CONNECTION` y `SSH_TTY` heredados están en blanco o ausentes. Un lanzamiento SSH conserva la línea de URL pero suprime la entrega al navegador porque el cliente SSH o el editor son dueños de la dirección local reenviada. Inmediatamente antes de una entrega, el runtime imprime `dsh web: opening the default browser; pass --no-open to disable`. Un helper de Node de corta vida ejecuta el opener de plataforma mantenido con el entorno hijo depurado canónico. En Windows permanece vivo hasta que el launcher de PowerShell de corta vida sale, porque `open` informa del spawn antes de que ese launcher haya entregado la URL a la shell; en el resto de plataformas el helper se detiene después de que el opener acepta el spawn. Un fallo del helper escribe un diagnóstico con su razón y la URL manual en stderr sin detener el servidor, y ningún camino espera a que el navegador salga. Este bundle también es dueño de la línea de comandos de la app: el provider ordinario `web-startup` ([`src/startup.ts`](src/startup.ts)) inyecta `ctx.cmdlineArgs` ([`dsh-cmdline`](../../boot/cmdline/README.es.md)), analiza `--host`, `--port`, el `--trusted-host` repetible, `--no-open` y el `--help` de la app, y luego aporta `webStartup`; la apertura del navegador está activada por defecto en los lanzamientos locales, y `--no-open` la desactiva para esta invocación. Rechaza `--host 0.0.0.0` antes de publicar ese servicio porque el CLI deliberadamente todavía no soporta el binding a todas las interfaces. Las filas configuradas por flag inyectan el servicio y lo leen directamente de la configuración perezosa, de modo que nada vincula un puerto antes de la resolución de argumentos y `dsh --profile web --help` no inicia ningún servidor. [`dsh-headless`](../headless/README.es.md) es una superficie hermana sobre la misma base y no monta este bundle.

## Valores predeterminados de reintento de modelo

Web usa el valor predeterminado normal acotado compartido de cinco reintentos elegibles después de la petición inicial. La ruta `deepseek-official` y las rutas pi-ai añadidas por ajustes usan ese valor predeterminado cuando omiten `retryPolicy`; las políticas explícitas de provider siguen ganando. Web no añade ninguna sobrescritura de composición específica de reintentos, así que el mismo comportamiento de omisión se aplica a los profiles que no son Web.

## Experiencia de modelo

### Contexto de harness-source y web-surface

#### Lo que ve el modelo

Cuando `surfaceContext` es true, la sección `harness:source` identifica la implementación de Harness en disco sin afirmar que es el directorio de trabajo, y la sección global `app:web-surface` (orden −98) orienta al modelo hacia la GUI: la URL local canónica, el referente «esta página», el contrato de actualización (el receptor de recarga está siempre activo; las recargas sin refresco necesitan además el watcher `pnpm run dev:web`) y la instrucción de no iniciar servidores de reemplazo. `DSH_WEB_URL` además aparece en el entorno bash gestionado con su descripción, resuelta por invocación desde el servidor en vivo. Cuando es false, no se registra ni la sección ni la variable.

#### Efecto de tokens

Una línea de fuente y un párrafo de prompt por sesión más dos líneas de variables de entorno gestionado; constante por proceso.

#### Efecto de KV Cache

La sección de prompt se sitúa cerca del inicio del system prompt y es estable durante toda la vida del proceso (el puerto es un hecho de arranque), de modo que no invalida la caché entre turnos.

## Limitaciones conocidas y trabajo diferido

- **El dist de frontend debe estar compilado** — el `require.resolve` del dist falla en voz alta en la activación con una pista de compilación; no hay respaldo de servir desde la fuente.
- **`lanAddresses` es una instantánea del arranque** — los cambios de interfaz posteriores al arranque no se vuelven a anunciar; la URL LAN impresa siempre coincide con la valla de confianza configurada.
- **Solo el inicio de la entrega es observable** — la observación termina cuando el opener de plataforma acepta el spawn, salvo que Windows espera a que su launcher de PowerShell de corta vida salga; una salida posterior del navegador no se informa, y la URL impresa sigue siendo el respaldo manual.
- **El reenvío SSH es dueño de la URL del navegador** — la URL canónica impresa nombra el endpoint de loopback del host remoto; la entrega automática se suprime, y el cliente SSH o el editor deben exponer y abrir su dirección local reenviada.
- **Las sobrescrituras del comando de navegador son solo de lanzamiento** — un `.env` descubierto no puede fijar `BROWSER`; solo un valor heredado puede llegar a una ruta de opener que honre la variable, así que un checkout no puede elegir un ejecutable para la entrega automática.
