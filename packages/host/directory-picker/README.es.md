# @deepseek-ai/dsh-host-directory-picker

[English](README.md) | Español

El selector de directorios de workspace del host de la GUI web es un capability seam. El servicio abstracto `DirectoryPicker` (`ctx.directoryPicker`) es su Service Definition. Su único método, `capability()`, devuelve una unión discriminada que describe cómo selecciona un directorio un operador. Los backends difieren en la interacción con el usuario, no solo en la implementación: `{ kind: 'native', pick(signal) }` abre un selector nativo del sistema operativo en la pantalla del host ([`-native`](../directory-picker-native/README.es.md)); `{ kind: 'browse', list(path?), createDirectory(path, name) }` ofrece operaciones de listado y creación para un navegador dentro de la aplicación, que funcionan para clientes remotos que no pueden llegar a un selector del sistema operativo ([`-browse`](../directory-picker-browse/README.es.md)). Los Consumers conmutan según `capability().kind`; la unión deriva del mapa `DirectoryPickerCapabilities`, extensible por fusión de declaraciones, y un backend nuevo añade ahí su variante. Ante un kind desconocido, los consumers ocultan la selección de directorios en lugar de fallar. El objeto de capacidad debe ser estable durante toda la vida del servicio. Cada paquete de backend tiene además un entrypoint de navegador que registra la interacción correspondiente en los slots de flujo de directorios de ui-workspace, de modo que una fila de composición selecciona a la vez la capacidad del host y el flujo del cliente. Una composición que deba elegir en runtime monta [`-auto`](../directory-picker-auto/README.es.md), que inspecciona el host una vez en el arranque y monta la fila de backend correspondiente.

Las primitivas de browse fallan con el error tipado `DirectoryPickerError` (`directory-unreadable` / `directory-exists` / `directory-create-failed`, cada uno con su `path` como sujeto), que el gateway consumidor mapea 1:1 a códigos de error de wire. Las filas de `DirectoryEntry` llevan una bandera `hidden` propiedad del host (convención del punto de POSIX) para que la política de visualización siga siendo del lado del cliente; `DirectoryListing.crumbs` es la cadena de ancestros desde la raíz del sistema de archivos, y cada miga es un destino de salto. El razonamiento de diseño, la separación de `ctx.fs` y las decisiones de política viven en [la Agent Note del capability seam de directory-picker](../../../.agents/notes/implemented/architecture/2026-07-28-directory-picker-capability-seam.es.md).

## Experiencia del modelo

Ninguna, ya que el seam atiende la selección de directorios del host de la GUI; nada de esto llega a una solicitud de modelo.

#### Efecto en la caché KV

Ninguno; este paquete ni ensambla ni envía una solicitud de provider.

## Limitaciones conocidas y trabajo diferido

- **Sin soporte multi-raíz** — el contrato de browse expone una sola cadena de ancestros por listado; el ámbito de raíz por despliegue (y la enumeración de raíces de unidad de Windows por encima de una unidad) espera a un consumer que lo necesite, según la Agent Note de DirectoryPicker.
