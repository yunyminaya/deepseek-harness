# @deepseek-ai/dsh-typert-protocol

[English](README.md) | Español

Declaraciones independientes del compilador compartidas por los paquetes de negocio, los artefactos Typert generados, el Host Gateway y la API de Cliente. Este paquete es dueño de la base de Remote Service, los decoradores, el fallback de binding explícito, los mapas de protocolo extensibles por fusión, los descriptores de invocación, los codecs y los contratos de provider; no ejecuta análisis TypeScript ni registra un servicio Cordis concreto.

## Declaraciones Remote

- `@Remote` marca un método público de instancia para su invocación directa en su Cordis Service registrado.
- `@RemoteScope(key)` marca un método cuyo receptor se selecciona de un tipo de Context con ámbito declarado por fusión.
- `TypertRemoteService` enlaza la clave de Cordis pasada a `super(ctx, serviceKey, options?)` al mismo namespace de wire por defecto.
- `bindTypertRemote(this, serviceKey, options?)` proporciona el mismo binding visible y congelado para un Service que no puede heredar de `TypertRemoteService`.
- `remoteMethods(service)` devuelve una instantánea desacoplada en orden de declaración que usa el fallback SRC del Gateway.

Un método de Host opta por la cancelación cooperativa declarando `signal: AbortSignal` como su último parámetro. `InvocationDescriptor.cancellation` registra ese punto de inyección reservado; la señal nunca se convierte en un parámetro JSON ni en un campo de lookup. SRC reconoce el nombre del último parámetro, mientras que la generación estricta verifica además el tipo global `AbortSignal`.

Los initializers de los decoradores conservan los marcadores en un `WeakMap` privado del módulo con clave el prototipo del Service. No añaden símbolos de constructor, propiedades de prototipo, metadatos de parámetros ni campos de reflexión en runtime. Un `TypertRemoteService` expone el mismo binding público de solo lectura `typertRemote` que devuelve el helper explícito.

## Protocolo Typert

Los paquetes de negocio extienden `TypertLookupMap` y `TypertContextMap` para asociar objetos de Host o Contexts con ámbito a sus identidades de wire. Los artefactos generados extienden `TypertRemoteMap`, `TypertRemoteScopeMap` y `TypertRemoteNamespaceMap` para que los imports del Cliente expongan solo los métodos Remote seleccionados. `InvocationDescriptor` es la forma de runtime compartida que consumen el registro, el Gateway y el Client Remote.

El ensamblado de Host extiende `TypertRemoteEventSelection` con los eventos de Host que reenvía a los consumidores, lo que estrecha la cara de claves de `ctx.remote.$on`; `TypertForwardableEvent` declara las formas que una entrega unidireccional puede transportar, excluyendo los eventos ligados a Scope y los eventos respondidos. `TypertClientRemote` desempeña ambos roles de esa superficie: los consumidores se suscriben mediante `$on`, y la mitad de Cliente que posee el sumidero de frames del host entrega los frames mediante `$dispatch`.

Los paquetes de Lookup y de Context son dueños de ambas caras de su contrato: la fusión de declaraciones aporta la asociación estática, mientras que los providers de runtime registran la resolución de identidad con `ctx.typert`. Un provider de Lookup o de Context de Host aporta la declaración estable y el resolver por defecto, mientras que la composición de Host puede configurar por separado un resolver síncrono o asíncrono; los rechazos de política pueden usar `TypertLookupFailure` para transportar un valor de fallo propiedad del adaptador de frontera. Los codecs estrictos transportan schemas generados; los codecs `src-json` identifican la vía más débil de arranque desde fuente.

## Experiencia del modelo

Ninguna, ya que este paquete de protocolo declara reflexión de aplicación y no registra nada orientado al modelo.

#### Efecto de KV Cache

Sin efecto directo.

## Limitaciones conocidas y trabajo diferido

- Los marcadores de los decoradores contienen solo el nombre del método y el modo de invocación directo o por Context. La reflexión de parámetros, resultados, lookups y schemas requiere la cadena de compilación Typert.
- Los decoradores Remote aceptan solo métodos públicos de instancia no estáticos con nombres de cadena. La ejecución SRC no puede representar firmas sobrecargadas, desestructuradas, con valores por defecto ni con parámetros rest.
