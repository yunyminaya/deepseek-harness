# Agent Note: Inyección de declaración de slot y ciclos de vida de recarga

Status: implemented

[English](2026-08-05-slot-declaration-injection.md) | Español

## Problema

Los plugins de cliente pueden contribuir a un slot antes o después del plugin que lo declara. La inyección de servicios de Cordis no puede expresar esta dependencia: un servicio es solo una señal de orden indirecta, las filas de dependencias del manifest de cliente no secuencian la activación, y un slot puede desaparecer y reaparecer mientras todos los servicios relacionados siguen montados. Registrar de inmediato compite por tanto con un slot no declarado, mientras que esperar a un servicio no relacionado acopla funcionalidades recargables de forma independiente.

El reemplazo en caliente a nivel de slot requiere también dos propietarios independientes. Quitar el plugin declarante debe quitar todas las contribuciones bajo sus slots hijos; quitar un plugin contribuyente debe quitar solo las entradas de ese plugin. Una declaración de reemplazo con la misma clave es un ciclo de vida nuevo incluso cuando la desaparición y la reaparición se agrupan en una sola notificación.

## Decisión

`SlotRegistry.inject(name, callback)` convierte el propio slot declarado en la dependencia. La clave completa de `SlotMap` se comprueba estáticamente; no hay constructor de espacios de nombres (namespace), servicio Cordis sintético ni `Context` específico de slot. El callback se ejecuta de inmediato cuando la declaración existe; en caso contrario espera, y devuelve o bien un disposer síncrono o bien un iterable síncrono de disposers. Los efectos iterables se instalan transaccionalmente: un fallo de configuración posterior libera (dispose) todos los efectos producidos antes, en orden inverso.

El ledger registra una epoch (época) de declaración distinta de la versión ordinaria de entrada del slot. Una epoch cambia cada vez que una declaración hija se crea o se colapsa. La inyección recuerda la época activa, libera el efecto de su callback cuando esa época termina y vuelve a ejecutar el callback para una declaración de reemplazo incluso cuando el estado final observado permanece declarado de forma continua. Los cambios ordinarios de contribución no reinician la inyección.

Ambos lados conservan su propiedad natural. El controlador de inyección y cada contribución se ejecutan en el `Context` del llamador del plugin contribuyente, de modo que liberar ese plugin elimina su espera y sus entradas activas. La cascada de colapso de hijos existente del ledger de slots elimina las entradas cuando el declarante desaparece; la inyección ejecuta entonces sus disposers para liberar los recursos de la capa de servicios y permanece lista para una declaración posterior. El `Context` del plugin declarante no se retiene como fuente de capacidades ni se expone a los contribuyentes.

El código de recarga dinámica usa una fibra de plugin de Cordis ordinaria como unidad de reemplazo: activa el módulo nuevo a través de `ctx.plugin()`, libera la fibra antigua y espera a que termine antes de montar su reemplazo, y deja que sus efectos `slots.inject` y `slots.register` se vayan con esa fibra. Las suscripciones del renderizador observan la eliminación del ledger y desmontan el componente; no se requiere ningún árbol de fibras propiedad del slot.

## Contrato de fallo y ciclo de vida

Una inyección cuya declaración ya existe notifica los fallos de configuración del callback de forma síncrona. Un fallo del callback tras una declaración diferida primero cancela la suscripción y revierte (rollback) sus efectos recopilados, y notifica después el fallo fuera del vaciado de notificaciones del slot, para que un registrante no pueda acaparar las notificaciones de otros listeners. El `slots.register()` directo sobre un slot no declarado sigue lanzando: la inyección es explícita y no debilita la validación en tiempo de carga.

Liberar una inyección es idempotente. Cancela la suscripción antes de liberar el efecto activo del callback, impidiendo que las notificaciones del ledger disparadas por el teardown resuciten la contribución. El teardown ligado a la declaración es síncrono con la frontera del ledger, de modo que libera los recursos de la capa de servicios antes de cualquier registro posterior en el mismo tick. Una inyección en espera liberada junto con su plugin no puede activarse más tarde.

## Alternativas consideradas

**Usar `ConversationController` u otro servicio como barrera de orden.** La presencia del servicio no identifica la declaración ni sigue su ciclo de vida de recarga, y crea una dependencia de paquete falsa para los contribuyentes solo de presentación.

**Convertir cada declaración en un servicio Cordis `slot:<name>`.** Esto contamina el espacio de nombres de servicios, convierte una clave dinámica mal escrita en una espera silenciosa de servicio y disfraza el estado del ledger como una capacidad de negocio. La inyección nativa de slots proporciona la misma espera sin cambiar la topología de Cordis.

**Crear un contexto o una fibra de Cordis para cada slot.** Un contribuyente necesita la intersección de su propio ciclo de vida de plugin y el ciclo de vida de la declaración, no las capacidades del declarante. Un contexto propiedad del slot introduce herencia de capacidades y problemas de teardown de doble padre sin mejorar la propiedad del ledger.

**Hacer que `register()` espere implícitamente.** El fallo inmediato sobre un destino no declarado es una comprobación de configuración valiosa. La inyección explícita distingue una contribución intencional ordenada de forma independiente de una composición rota.

**Juzgar el reemplazo solo por `spec(name) !== undefined`.** El colapso y la redeclaración pueden agruparse en un estado final continuamente presente mientras las contribuciones antiguas ya se han eliminado. La epoch de declaración preserva esa frontera.

## Consecuencias

Las dependencias de slots se vuelven auditable en el punto de registro y siguen el reemplazo de declaraciones sin convenciones de orden específicas de paquete. La liberación dinámica de plugins elimina las entradas renderizadas a través de los efectos Cordis existentes, mientras que el reemplazo de declaraciones tiene un hook estable para el HMR a nivel de slot posterior.

El runtime conlleva una epoch monótona adicional por cada slot tocado, y los callbacks de inyección deben devolver su limpieza. Los callbacks de registro múltiple usan efectos iterables para que la configuración y el teardown sigan siendo atómicos. El ledger plano de claves con puntos y la autoridad única de composición de `register()` permanecen sin cambios.
