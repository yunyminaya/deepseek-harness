# @deepseek-ai/dsh-typert-loader

[English](README.md) | Español

Integración del Loader solo para Node con los artefactos Typert generados. El plugin requiere `ctx.loader` y `ctx.typert`; no proporciona el registro en sí.

Durante la activación escanea las entradas existentes del Loader. Luego sigue las notificaciones de ciclo de vida `internal/plugin` de Cordis, resuelve el `package.json` de cada paquete de entrada, importa `./typert` cuando está exportado, valida su manifest `TYPERT` y registra la contribución hasta que la entrada o este plugin se desmonten. Un import que se resuelve después de que cualquiera de los dos propietarios haya desaparecido se descarta.

`packages` lista artefactos de paquete adicionales que registrar para plugins anidados detrás de otra entrada del Loader. Las fibers de Cordis no conservan los especificadores npm de esos plugins anidados, por lo que esta frontera es explícita; cada paquete configurado debe resolverse desde el árbol de configuración y exportar `./typert`.

Los paquetes sin la exportación se omiten. La resolución de paquetes y los manifests importados se almacenan en caché durante toda la vida del proceso, por lo que añadir una exportación exige reiniciar. Un artefacto malformado hace fallar la activación cuando ya está montado; un fallo posterior se registra sin impedir que paquetes no relacionados se registren.

## Experiencia del modelo

Ninguna, ya que el loader solo alimenta [`ctx.typert`](../registry/README.md); los consumidores son dueños de cualquier proyección visible para el modelo.

#### Efecto de KV Cache

Sin efecto directo.

## Limitaciones conocidas y trabajo diferido

- El descubrimiento importa solo la cara de host; los runtimes de cliente necesitan un propietario de composición aparte antes de añadir un descubrimiento equivalente.
- Las entradas del Loader se descubren automáticamente. Los plugins anidados o ajenos al Loader requieren una entrada `packages` explícita o la propiedad directa de `ctx.typert.register()`.
