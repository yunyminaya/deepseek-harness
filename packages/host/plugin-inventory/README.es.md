# @deepseek-ai/dsh-host-plugin-inventory

[English](README.md) | Español

Proyección de solo lectura del Host del árbol actual del Loader de Cordis. `PluginInventoryGateway` registra el servicio `pluginInventory` y publica un Remote directo generado, `pluginInventory/list`. Cada llamada lee `ctx.loader.entries()` directamente, omite las filas de grupo estructurales y devuelve las entradas restantes en orden del Loader con solo su id de entrada del Loader, su especificador de módulo, su habilitación efectiva y la fase actual de su Fiber raíz.

La fase es `pending`, `loading`, `active`, `failed` o `unloading`; es `null` cuando la entrada no tiene un Fiber raíz vivo. La instantánea es deliberadamente puntual: el Loader sigue siendo la única autoridad de ciclo de vida, mientras que este paquete no posee caché, historial, modelo de procedencia, flujo de eventos ni vía de mutación. Sus tipos de payload públicos viven en `./types`, y Typert genera los artefactos Remote del Host y del Cliente que exponen `./typert` y `./remote`.

El servicio es solo Remote y deliberadamente no declara ninguna fusión de `Context` de Cordis en el mismo proceso. Los paquetes de cliente lo consumen a través del ensamblaje explícito [`api-remotes`](../../api/remotes/README.es.md) en lugar de importar la implementación del Host.

## Experiencia del modelo

Ninguna, ya que esta proyección de inventario solo-Host no registra ningún prompt, herramienta, mensaje ni solicitud de provider.

#### Efecto en la caché KV

Ninguno; este paquete nunca ensambla entrada de modelo.

## Limitaciones conocidas y trabajo diferido

- **Solo estado puntual** — el resultado no contiene historial de fallos duradero ni suscripción; un Fiber raíz ausente se informa como `null`, sea cual sea la razón por la que no existe una raíz viva.
- **Sin procedencia ni mutación** — el servicio no identifica qué bundle, perfil u override introdujo una entrada y no puede habilitar, deshabilitar, añadir ni eliminar plugins.
