# @deepseek-ai/dsh-client-ui-directory-picker-native

[English](README.md) | Español

Superficie de selector de directorio nativo: la mitad de navegador de la interacción de selección nativa. Llena los dos huecos de flujo de directorios de ui-workspace (`conversation.hero.workspace.directoryFlow` y `sidebar.workspaces.directoryFlow`) con un ocupante sin renderizado que responde a cada petición `open` manejando el selector del sistema operativo del Host local a través de `ctx.workspaces.pickDirectory()` y luego informa exactamente de un resultado —una ruta elegida, una cancelación o un fallo— de vuelta a través de la conversación del propietario. El propio diálogo del sistema operativo pertenece a [`dsh-host-directory-picker-native`](../../host/directory-picker-native/README.es.md); montar este paquete compone la superficie con ese backend desde una sola fila de cordis.yml, de modo que ningún código de cliente se bifurca según un tipo de capacidad.

Ambos registros se instalan como un único efecto transaccional mediante llamadas anidadas de `slots.inject()`, porque cualquiera de las entradas declarantes puede activarse más tarde o sustituir su declaración. El ocupante se arma una vez por cada flanco ascendente de `open`, así que los re-renderizados —incluida una adopción que mantiene `open` en true mientras `busy`— nunca lanzan un segundo selector, y que el propietario retire `open` vuelve a armar la siguiente petición. Las resoluciones viajan en una ref para que la respuesta llegue a los manejadores más recientes del propietario y no a los capturados cuando se abrió el selector. Un desmontaje (HMR que sustituye al ocupante) descarta la resolución por completo: el protocolo no admite cancelación por petición, así que el selector del lado del Host sobrevive hasta que se responde, su respuesta no llega a ninguna parte y la instancia de reemplazo se vuelve a armar bajo la petición aún abierta del propietario.

La mitad Node es un `apply` vacío: existe para que el plugin aparezca en el cordis.yml del host y en el Loader, mientras que la mitad de navegador se distribuye a través de `exports["./client"]` y se descubre mediante la declaración de manifest `dsh.client`.

## Experiencia del modelo

Ninguna, ya que el selector de directorio es chrome del navegador; nada de esto llega a una petición de modelo.

#### Efecto en la KV Cache

Ninguno; este paquete ni ensambla ni envía una petición de provider.

## Limitaciones conocidas y trabajo pendiente

- **Sin cancelación de un selector abierto** — el protocolo no admite cancelación por petición, así que un selector ya en la pantalla del Host no puede cerrarse desde el navegador; una resolución descartada simplemente se ignora.
- **Solo en Hosts locales** — un diálogo del sistema operativo se abre en la máquina que ejecuta el Host, así que los despliegues en proceso y de navegador remoto necesitan en su lugar la composición `-browse`. Los fallos de plataforma afloran a través del diálogo de carpeta reintentable del propietario.
