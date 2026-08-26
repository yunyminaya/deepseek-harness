# @deepseek-ai/dsh-typert-registry

[English](README.md) | Español

Registro de runtime para los artefactos Typert generados. Una contribución transporta la reflexión de negocio de una cara de paquete y schemas Zod vivos opcionales; `ctx.typert` registra ambos atómicamente y los retira con la fiber de Cordis que llama. El análisis TypeScript y la generación de código viven en [`dsh-typert-generator`](../generator/README.md).

La reflexión de paquete se indexa por `<package>#<face>`. Los schemas se indexan por `<package>#<name>` y conservan la instancia de Zod del productor. El JSON Schema se calcula bajo demanda en el borde del consumidor.

## API pública

- `TypertRegistry` es el plugin por defecto y proporciona `ctx.typert`.
- `ctx.typert.lookups.register()` registra la declaración de wire y el resolver por defecto propiedad del paquete de negocio; `configure()` registra un resolver propiedad de la composición de Host que puede ejecutarse de forma asíncrona. Sus ciclos de vida son independientes: la configuración puede preceder al provider, y descargar la configuración restaura la política por defecto.
- `ctx.typert.contexts.registerHost()` y `configureHost()` aplican la misma división de propiedad a la identidad de Context con ámbito; `registerClient()` aporta el binder de Context de Cliente correspondiente.
- `register(contribution)` rechaza identidades malformadas y claves duplicadas de cara-paquete o de schema antes de confirmar nada, y luego devuelve el disposer de efecto Cordis exacto.
- `get(key)`, `resolve(key)` y `list(filter?)` consultan los schemas vivos. `resolve()` distingue una clave malformada, un paquete ausente y un paquete que no contribuye ningún schema con ese nombre.
- `getPackage(packageName, face?)` y `listPackages(filter?)` consultan la reflexión generada de servicios, eventos y objetos; la cara por defecto es `host`.
- `toJSONSchema(key, params?)` proyecta un schema vivo con `z.toJSONSchema()` sin almacenar en caché el resultado.
- `typertKey()` y `typertPackageKey()` componen las dos formas estables de identidad.

El subpath `@deepseek-ai/dsh-typert-registry/types` contiene los contratos puros de contribución y de registro. [`dsh-typert-loader`](../loader/README.md) descubre y registra los artefactos de host generados en composiciones del Loader; el `ctx.typert.register()` directo da soporte a otros propietarios de composición.

## Experiencia del modelo

Ninguna, ya que el registro no contribuye ningún prompt, herramienta ni evento de sesión; consumidores como `cordis_inspect` son dueños de cualquier proyección visible para el modelo.

#### Efecto de KV Cache

Sin efecto directo. Un consumidor que coloque reflexión en una solicitud es dueño del cambio de prefijo resultante.

## Limitaciones conocidas y trabajo diferido

- El registro almacena la reflexión generada pero no fusiona los grafos de host y de cliente ni resuelve referencias TypeScript. Esas son preocupaciones del analizador y del emisor.
- Las claves de schema omiten la cara porque host y cliente se ejecutan en contextos separados. Registrar schemas con el mismo nombre de ambas caras en un contexto se rechaza como duplicado.
