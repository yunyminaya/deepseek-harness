# Agent Note: Source-owned session immutability and dev-mode invariants

Status: implemented

[English](2026-06-11-dev-invariants-over-deep-readonly.md) | Español

## Problem

El registro de sesión necesita dos protecciones diferentes: propiedad inmutable de cada hecho almacenado y verificaciones de relaciones entre hechos a través del tiempo y contratos de servicio. Confluirlos en un plugin de desarrollo opcional dejaría el historial de producción vulnerable; intentar expresar ambos a través de tipos readonly de TypeScript no crearía un límite en tiempo de ejecución ni describiría reglas relacionales.

El registro de sesión es la fuente duradera de verdad para replay, reconstrucción de solicitudes, persistencia e historial visible al usuario. El código fuera del paquete de sesión debe poder inspeccionar ese historial sin retener una referencia que pueda reescribirlo más tarde, y las entradas aceptadas de los llamadores no deben permanecer conectadas a objetos mutables propiedad del llamador.

La inmutabilidad de valores individuales es solo la mitad del contrato. Un registro puede contener registros perfectamente inmutables cuya secuencia, anidamiento de turno/paso, emparejamiento de tool-call, entrega con ámbito o solicitud de modelo reconstruida es incorrecta. Esas reglas relacionan múltiples registros o servicios y no pueden establecerse congelando un solo objeto.

Los tipos readonly de TypeScript no son un límite de tiempo de ejecución suficiente. Desaparecen cuando el programa se ejecuta, un cast puede evitarlos y un `DeepReadonly<T>` recursivo se propagaría a través de cada consumidor de registro y mensaje aunque algunas APIs de procesamiento de solicitudes posteriores trabajen intencionalmente con valores mutables.

## Decision

La responsabilidad se divide entre un límite de almacenamiento siempre activo y afirmaciones de desarrollo opcionales.

### Session owns immutable history

`Session` acepta un evento solo después de que un pase recursivo haya materializado una instantánea JSON sin pérdida. Ese pase rechaza valores no admitidos y produce el registro separado exacto que entra en el registro, por lo que la validación y el almacenamiento no pueden observar valores diferentes de un getter con estado o retener referencias anidadas propiedad del llamador.

El evento aceptado y todos sus descendientes se congelan profundamente antes de la publicación. `append()` devuelve ese evento congelado propiedad de la sesión, los observadores de `session/event` reciben el mismo registro, y `session.events` devuelve una instantánea de array congelada. Un array previamente devuelto no crece después de un append posterior. Los registros semilla pasan por el mismo límite de validación, instantánea y congelación antes de que la construcción tenga éxito.

Esta garantía pertenece a `Session`, no a un listener opcional, porque cada composición confía en historial confiable. Un despliegue de producción, una prueba enfocada o una inserción personalizada reciben las mismas semánticas de almacenamiento tanto si los plugins de soporte de desarrollo están registrados como si no.

### Derived requests remain detached

`deriveMessages()` proyecta eventos de superficie registrados en objetos `Message` separados y congelados profundamente y devuelve una instantánea de array nueva. El ensamblaje de solicitudes por lo tanto puede combinar historial derivado con otras entradas sin exponer una ruta de regreso al registro. El caché reutiliza proyecciones inmutables seguras en lugar de re-clonar el historial completo para cada llamada de modelo.

### Package-owned invariant companions check relationships

`dsh-invariants` registra el servicio configurable `ctx.invariants` y no contiene verificaciones de producto. Cada paquete publica un compañero de propiedad `./invariant`; `dsh-session`, `dsh-agent`, `dsh-scope` y `dsh-agent-loop` actualmente agregan las reglas que requieren estado de traza u observación de otro seam: números de secuencia monótonos, anidamiento de turno y paso, emparejamiento tool-call/result, transiciones de estado de agente legales, despacho con ámbito correcto por asunto e igualdad entre una solicitud construida por el loop y la solicitud reconstruida desde su prefijo de registro de sesión. La habilitación global y filtros de regex por nombre de paquete pertenecen al servicio ([servicio invariants propiedad de paquete](2026-07-19-package-owned-invariant-service.es.md)).

Cuando el compañero de sesión se adjunta a una sesión existente o semilla, repite el registro inmutable para reconstruir el estado de traza. El servicio da a cada contribución una fiber hija desechable, por lo que la recarga en caliente es segura a mitad de un turno sin dar a los diagnósticos propiedad del almacenamiento de sesión.

## Alternatives considered

### Pervasive deep-readonly types

Una propuesta de compañero rechazada aplicaría un tipo recursivo `DeepReadonly<T>` en las superficies públicas de registro y mensaje, volviendo las rutas de lectura de sesión (`events`, listeners de `session/event`, `deriveMessages()`) a deep-readonly mientras mantiene las cascadas en vuelo mutables. Eso provee retroalimentación del editor pero no una garantía en tiempo de ejecución: los tipos de TypeScript se borran y el código de plugin puede hacer cast a través de ellos. También empuja tipos readonly a consumidores donde la mutación es intencional. La propiedad en tiempo de ejecución en el límite `Session` protege a cada llamador sin esa propagación de tipos.

### Development-only freezing

Congelar el historial solo cuando un plugin invariants está instalado haría la garantía central dependiente de la composición. El código podría pasar pruebas de desarrollo y aún corromper el historial en producción o en una composición enfocada que omite el plugin. La inmutabilidad de almacenamiento por lo tanto está siempre activa, mientras las verificaciones relacionales más costosas permanecen como soporte de desarrollo opcional.

### Clone only when deriving messages

Separar `deriveMessages()` protegería la ruta de solicitud más común pero dejaría otros lectores de `session.events`, valores de retorno de append y observadores de eventos de sesión capaces de mutar el historial duradero. El registro debe proteger su propio límite; las proyecciones derivadas son un límite de aislamiento adicional, no un sustituto.

## Consequences

- Cada evento de sesión en vivo o semilla aceptado está separado de las entradas propiedad del llamador y es profundamente inmutable antes de que cualquier observador pueda recibirlo.
- `session.events` expone instantáneas inmutables estables en lugar del array privado en crecimiento.
- La mutación del lado de solicitud no puede alcanzar el historial almacenado a través de mensajes derivados.
- Las compilaciones de desarrollo pueden habilitar afirmaciones relacionales sin cambiar el comportamiento de almacenamiento, y disponer o filtrar un compañero no debilita la inmutabilidad del registro.
- `dsh-invariants` configura habilitación global más listas de regex allow/block por nombre de paquete; cada verificación permanece propiedad y probada por su paquete de producto.
- El límite en tiempo de ejecución lleva un costo de instantánea y congelación recursiva una vez por evento aceptado; los lectores posteriores y proyecciones en caché reutilizan los registros inmutables propiedad.