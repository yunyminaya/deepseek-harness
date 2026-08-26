# @deepseek-ai/dsh-api-gateway

[English](README.md) | Español

Endpoint RPC de Typert de doble cara para los entornos Cordis Host y Client. La entrada Host proporciona `ctx.typertGateway`, mientras que `@deepseek-ai/dsh-api-gateway/client` proporciona `ctx.remote`; ambas consumen el mismo contrato generado `InvocationDescriptor` y dejan la selección de negocio a API Remotes, y el transporte, la correlación de solicitudes, la confianza y los envoltorios de respuesta a Connection.

## Servicio Host: `TypertGatewayService` (clave de ctx: `typertGateway`)

`ctx.typertGateway.invoke()` resuelve el descriptor actual y el Service de Cordis para cada llamada, valida los argumentos con nombre exactos, resuelve las identidades registradas de objeto o Context, invoca el método de negocio público y valida su resultado. Los Business Services extienden `TypertRemoteService` y marcan los métodos con `@Remote` o `@RemoteScope` de [`dsh-typert-protocol`](../../typert/protocol/README.es.md); `bindTypertRemote()` sigue disponible cuando otra clase base es dueña de la herencia.

El modo estricto lee los descriptores de invocación generados de `ctx.typert.local`. Los parámetros de búsqueda usan el resolver activo actualmente en `ctx.typert.lookups`: el paquete de negocio registra la declaración estable y la política por defecto, mientras que la composición del Host puede anular el comportamiento de resolución con `configure()` con ámbito de efecto; `@RemoteScope` resuelve su receptor a través de un provider de Context del Host registrado. El modo SRC es un respaldo de desarrollo para endpoints que nunca han tenido una definición estricta; analiza nombres de parámetros simples y solo acepta valores seguros para JSON en los parámetros que no son de búsqueda. Retirar una definición estricta observada falla en lugar de debilitar la validación.

La entrada Host registra un interceptor de host de confianza en el FetchHandler `/api` compartido de Connection. Connection pasa este handler compuesto a través de su puente HTTP; el handler despacha los endpoints reclamados a Gateway y los no reclamados al API Proxy. Las llamadas directas a `invoke()` conservan los errores de negocio; `TypertGatewayError` distingue los fallos propiedad del despacho, el enlace, los providers, la búsqueda, el Context, los argumentos y los codecs. Un resolver puede usar `TypertLookupFailure` para transportar un error RPC existente, conservando su código de error original para rechazos de política como los fallos de reanudación en frío o las vallas de propiedad.

Un método Remote consciente de la cancelación declara `signal: AbortSignal` como su último parámetro Host. La señal es metadatos de descriptor, no un argumento de cableado: Connection se la proporciona al Gateway, y el Gateway la inyecta después de los parámetros de negocio decodificados. SRC reconoce el último nombre reservado, mientras que la generación estricta exige además el tipo global `AbortSignal`.

## Servicio Client: `ClientRemote` (clave de ctx: `remote`)

`ctx.remote.$mount()` valida y registra una contribución generada de Host para Client y luego instala métodos concretos directos y con ámbito para la fiber de Cordis llamante. Cada espacio de nombres es un Service hijo trazado `remote.<namespace>` y se descarga después de retirar su último método. Los endpoints duplicados, las colisiones de espacios de nombres y los descriptores sin codecs generados estrictos fallan antes de que los métodos puedan llamarse.

Cada llamada valida las entradas posicionales, construye los `args` con nombre exactos del descriptor y los envía a través de `ctx.connection.rpc.call('/api', endpoint, ...)`. Los métodos generados conscientes de la cancelación aceptan un `AbortSignal` final opcional; el Client lo combina con la vida del montaje de la contribución antes de llamar a Connection. El valor devuelto se valida antes de llegar al código de la aplicación. Retirar una contribución elimina juntos sus descriptores y métodos, aborta las llamadas en vuelo y hace que los handles de método retenidos rechacen.

`ctx.remote.$on()` se suscribe a un evento del Host reenviado. Sus claves legales son exactamente la selección de reenvío del ensamblaje del Host, y el tipo de listener es la propia declaración `Events` de Cordis del paquete propietario, de modo que ninguna segunda firma puede alejarse de ella. Cada suscripción pertenece a la fiber llamante y desaparece con ella. La entrega es unidireccional y sigue el orden de registro; un listener que lanza una excepción se registra y se aísla del resto de listeners, lo que nunca afecta al bombeo de frames. `ctx.remote.$dispatch()` es la otra mitad de esa superficie, y es del portador: la mitad Client dueña del sumidero de frames del Host entrega cada frame decodificado, y un nombre de evento al que nadie se suscribe se descarta, porque el cableado transporta lo que el Host haya seleccionado. Un consumidor se suscribe y nunca lo llama.

Las fusiones de declaraciones generadas proporcionan la API de TypeScript a través del contrato compartido `TypertClientRemote`. La entrada Client no contiene ningún merge de interfaz de Host Service ni de Cordis del Host, y la búsqueda e invocación de métodos usan objetos y funciones ordinarios en lugar de un Proxy de JavaScript.

## Model Experience

Ninguna, ya que el paquete despacha llamadas de aplicación y no registra ningún evento de prompt, herramienta o sesión.

#### Efecto de KV Cache

Sin efecto directo; los Business Services invocados son dueños de cualquier resultado visible para el modelo.

## Limitaciones conocidas y trabajo diferido

- El adaptador de Connection mapea los fallos ordinarios de despacho y las excepciones de negocio al código `internal` de RPC con detalles vacíos; los errores de política de búsqueda transportados por `TypertLookupFailure` se devuelven sin cambios. Las categorías estructuradas de `TypertGatewayError` siguen disponibles solo para los llamadores del mismo proceso.
- El modo SRC admite parámetros de identificador único sin desestructuración, valores por defecto ni parámetros rest. Valida la seguridad JSON en lugar de los tipos de negocio generados y nunca infiere campos opcionales.
- Solo las contribuciones generadas estrictas pueden montarse en la cara Client. Los marcadores SRC no tienen codec de Client ni proyección de tipos.
- El paquete solo despacha métodos unarios. Los datos incrementales de sesión usan un protocolo separado de flujo con nombre sobre la misma Connection.
- Los resolvers de búsqueda se configuran por clave; un parámetro Remote o un endpoint concretos no pueden seleccionar por ahora una política solo en vivo bajo la misma clave `agent`/`session`.
- Los eventos reenviados llegan a `$on` exactamente como los emitió el Host: sin proyección ni censura de payload, sin suscripción ligada a Scope y sin replay tras una reconexión.
