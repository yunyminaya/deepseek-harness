# @deepseek-ai/dsh-host-directory-picker-native

[English](README.md) | Español

El **backend de selector nativo del SO** del [seam de directory-picker](../directory-picker/README.es.md): `NativeDirectoryPicker` registra `ctx.directoryPicker` con la capacidad `native`, cuyo `pick(signal)` abre un selector nativo del SO por llamada y resuelve la ruta absoluta elegida (`null` al cancelar). Las herramientas de plataforma corren sin shell: `osascript` en macOS y Zenity con fallback a KDialog en Linux; la anulación del llamador termina el proceso nativo. Windows abre el `IFileOpenDialog` moderno en un proceso hijo spawned — una conversación COM guiada por koffi en el hilo principal del hijo con la mejor conciencia DPI de hilo que el host acepte (per-monitor-v2 primero), abortada enviando `WM_CLOSE` al hilo del diálogo. Solo es viable cuando el operador está ante el display del host — los despliegues remotos componen [`-browse`](../directory-picker-browse/README.es.md) en su lugar. La frontera de comandos (`DirectoryPickerRunner`) y los hechos de plataforma son inyectables. El runner de subprocesos sin shell compartido vive en [`dsh-native-command`](../../util/native-command/README.es.md).

**Paquete dual-face**: la mitad de navegador (`./client`) registra un ocupante de flujo sin renderizado en los dos huecos de flujo de directorios de [ui-workspace](../../client/ui-workspace/README.es.md) — cada petición `open` conduce `host.pickDirectory` e informa del único resultado (ruta elegida / cancelación / fallo) a través de la conversación del propietario del hueco. Ambas declaraciones de flujo de directorios deben estar activas antes de que se instale cualquiera de las dos contribuciones. Una fila de cordis.yml compone por tanto ambos lados de la interacción nativa; el cliente no lleva ramificación por clase de capacidad, y montar un segundo paquete de flujo falla en la carga (los huecos son de kind `single`).

## Experiencia de modelo

Ninguna, ya que el backend sirve la selección de directorios del host con GUI; nada de aquí llega a una petición del modelo.

#### Efecto de KV Cache

Ninguno; este paquete ni ensambla ni envía una petición de provider.

## Limitaciones conocidas y trabajo diferido

- **Linux exige herramientas de escritorio** — sin Zenity ni KDialog instalados, `pick` rechaza con un error accionable; no cae a un prompt de ruta tecleada (el backend browse es ese fallback a nivel de composición).
- **Windows no tiene fallback de mecanismo** — el picker por proceso hijo a través del koffi empaquetado es el único nivel nativo, así que un rechazo de COM o un fallo del diálogo muestra el error. El backend browse sigue siendo el fallback a nivel de composición.
