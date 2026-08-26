# @deepseek-ai/dsh-client-ui-directory-picker-browse

[English](README.md) | Español

Superficie de exploración de directorios dentro de la aplicación: la mitad de navegador de la interacción de selección por exploración. Llena los dos huecos de flujo de directorios de ui-workspace (`conversation.hero.workspace.directoryFlow` y `sidebar.workspaces.directoryFlow`) con el diálogo Select Workspace Directory, manejando las primitivas `host.listDirectory` y `host.createDirectory` del Host local a través de `ctx.workspaces`. Su contraparte Node es [`dsh-host-directory-picker-browse`](../../host/directory-picker-browse/README.es.md); montar este paquete compone la superficie con ese backend desde una sola fila de cordis.yml, de modo que ningún código de cliente se bifurca según un tipo de capacidad. A diferencia de la superficie [`-native`](../ui-directory-picker-native/README.es.md), el diálogo no necesita un selector del sistema operativo local, así que también sirve para despliegues en proceso y de navegador remoto.

El diálogo es una vista de columnas Miller de 680×500 (limitada en viewports cortos o estrechos): una cabecera con el título, las migas de pan de la ruta de selección y una zona de ruta clicable para editar, luego un nivel de ancho completo hasta que se selecciona una fila, tras lo cual la fila se divide en partes iguales en columnas de nivel y de hijos. Las navegaciones aterrizan ancladas a la selección y en silencio —la vista anterior sigue renderizando mientras se escanea un salto de migas o una ruta enviada, y el tramo de destino y el del padre aterrizan en un mismo frame—, así que retroceder mantiene dos paneles de distancia respecto a la raíz de la vista y ningún frame intermedio parpadea. **Nueva carpeta** abre un diálogo de creación anidado dirigido a la carpeta seleccionada y selecciona lo que crea; **Abrir** adopta la carpeta seleccionada, con el nivel listado como alternativa. Las entradas ocultas marcadas por el Host permanecen ocultas hasta que el conmutador del pie las revela, lo que es solo un filtro del lado del cliente.

Confirmar un directorio es la ruta elegida y cerrar el diálogo es la cancelación. Los fallos de exploración —un destino ilegible, un conflicto de creación— permanecen en las superficies de alerta propias del diálogo, así que este ocupante nunca acciona la rama `onError` del propietario; el propietario conserva la superficie de error de creación de workspace. Ambos registros se instalan mediante llamadas anidadas de `slots.inject()` porque cualquiera de las entradas declarantes puede activarse más tarde o sustituir su declaración, y el texto del diálogo se registra en el propio espacio de nombres de locale de este paquete: los dos diccionarios aterrizan como una unidad, de modo que una activación fallida no puede ocupar en solitario un locale del espacio de nombres.

La mitad Node es un `apply` vacío: existe para que el plugin aparezca en el cordis.yml del host y en el Loader, mientras que la mitad de navegador se distribuye a través de `exports["./client"]` y se descubre mediante la declaración de manifest `dsh.client`.

## Experiencia del modelo

Ninguna, ya que el explorador de directorios es chrome del navegador; nada de esto llega a una petición de modelo.

#### Efecto en la KV Cache

Ninguno; este paquete ni ensambla ni envía una petición de provider.

## Limitaciones conocidas y trabajo pendiente

- **Sin búsqueda, sin multiselección y sin renombrar ni eliminar** — el diálogo lista y crea directorios; se llega a un destino navegando, editando la ruta o filtrando el último panel por prefijo.
- **El filtrado de entradas ocultas es del lado del cliente** — el Host siempre lista las entradas ocultas y las marca, así que el conmutador solo cambia lo que renderiza el diálogo.
