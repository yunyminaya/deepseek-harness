# Agent Note: el seam code-runtime es dueño de las exclusiones de identificador portable

Status: implemented

[English](2026-07-31-code-runtime-portable-identifier-seam.md) | Español

## Problema

El seam code-runtime promete que una lista de binding namespaces válida en un backend es válida en todos los backends, de modo que un Consumer de Code Mode puede entregar los mismos bindings a cualquier runtime registrado sin conocer su lenguaje. El primer backend, `dsh-code-runtime-worker-thread`, poseía en privado las reglas de identificador que hacen cumplir parte de esa promesa: una regex `IDENTIFIER` que admitía el `$` exclusivo de JS, un conjunto `RESERVED_WORDS` que contenía solo palabras clave de ECMAScript y un conjunto `RESERVED_ERROR_PROPERTIES` con tres slots de `Error` de JS. Esas reglas describían el propio lenguaje del worker, no el contrato de portabilidad del seam.

Un segundo backend escrito para un lenguaje distinto (CPython) tendría que o bien redeclarar sus propias reglas —dejando que `lambda` pase el worker y falle en Python, o que `$tools` pase el worker y falle en todo backend que no sea JS—, o bien importar las del worker, invirtiendo la dependencia de modo que un Service Provider alcanzara a un Service Provider hermano. Ninguna opción mantiene real la promesa de portabilidad: solo valdría para el backend contra el que un llamador hubiera probado por casualidad.

## Decisión

El paquete de Service Definition (`@deepseek-ai/dsh-code-runtime`) exporta el contrato de exclusiones de identificador portable como cuatro constantes con nombre, y todo Service Provider las importa en lugar de redeclararlas:

- `PORTABLE_RESERVED_WORDS` — la unión de las palabras reservadas de ECMAScript y Python. Un global de namespace o un nombre de clase de error que coincida con cualquiera se rechaza en todos los backends, de modo que `lambda` se rechaza aunque sea un nombre de parámetro legal en JS. Añadir un lenguaje amplía esta unión, lo que constituye una revisión deliberada y rompedora de los nombres de binding existentes.
- `RESERVED_BINDING_GLOBALS` — los globals que algún backend posee en el namespace del programa: `console` (la captura de logs del worker), `__dsh_main__`/`__builtins__`/`__name__` (el wrapper del bootstrap de Python y los globals de módulo sembrados) y `__debug__` (no es un slot sembrado, sino una constante de tiempo de compilación de CPython que rechaza la asignación, de modo que un global inyectado con ese nombre es inalcanzable —la misma división de portabilidad por un mecanismo distinto). Se rechazan en todas partes para que una lista de namespaces no pueda elegir un nombre que funcione en un backend y colisione en otro.
- `RESERVED_ERROR_MEMBERS` — los nombres de miembros de error que todo backend rechaza: los slots de `Error` de JS (`name`, `message`, `stack`) y los miembros del protocolo de excepciones de Python (`args`, `with_traceback`, `add_note`).
- `DUNDER_MEMBER` — la regex de la forma dunder (`__x__`, parte central no vacía), rechazada como miembro de error en bloque porque varios son descriptores de CPython restringidos cuyo conjunto exacto es un detalle de la versión del intérprete.

El Service Definition también reduce el subconjunto de identificadores portables a `[A-Za-z_][A-Za-z0-9_]*` (documentado en `CodeBindingNamespace.global` y `CodeBindingErrorClass`), eliminando el `$` exclusivo de JS. El worker consume las constantes compartidas directamente bajo sus nombres exportados —`PORTABLE_RESERVED_WORDS` tanto para nombres de binding global como de clase de error, `RESERVED_BINDING_GLOBALS` para los slots propiedad del backend, `RESERVED_ERROR_MEMBERS` más `DUNDER_MEMBER` para los miembros de error—, sin re-alias local; su regex `IDENTIFIER` pierde el `$`.

Las constantes viven en el Service Definition aunque el worker sea el único backend publicado: el sentido mismo es que el contrato es agnóstico del lenguaje y es propiedad de algo por encima de cualquier lenguaje individual. Un Service Provider que lo violara sería el bug, y el conjunto compartido es donde un revisor mira para ver qué significa «portable».

## Alcance

Esta decisión entrega solo la extensión del Service Definition y la adopción de esta por parte del worker. El renderizador `py-types` y el despacho de lenguaje de Code Mode son propiedad de la [nota de despacho de lenguaje](../feature/2026-07-31-code-mode-language-dispatch.es.md); un backend de Python todavía no existe. El README del Service Definition conserva su redacción de solo-worker por esa razón: enlazar a un README de `dsh-code-runtime-python` que no existe rompería el gate de enlaces muertos.

`RESERVED_BINDING_GLOBALS` codifica el diseño concreto del bootstrap de Python antes que el propio backend: siembra exactamente `__builtins__`/`__name__` y envuelve el programa bajo `__dsh_main__`. Un backend de Python que siembre cualquier global de módulo adicional (`__doc__`, `__loader__`, `__spec__`, `__file__`, `__package__`, …) DEBE (MUST) ampliar este conjunto en el mismo cambio, exactamente igual que añadir un lenguaje amplía `PORTABLE_RESERVED_WORDS` —un nombre que el bootstrap siembra pero que el conjunto omite es la división de portabilidad que este contrato existe para prevenir.

## Alternativas consideradas

**Cada backend declara sus propias exclusiones.** Rechazado: hace que la promesa de portabilidad sea por backend. Una lista de bindings que el llamador probó en el worker podría ser rechazada por Python, que es exactamente la división que el seam existe para prevenir.

**El backend de Python importa las constantes del worker.** Rechazado: invierte la dependencia —los Service Providers del seam alcanzarían una implementación hermana para obtener un contrato que ninguno posee. El contrato pertenece a algo por encima de ambos, al seam.

**Mantener `$` en el subconjunto de identificadores portables.** Rechazado: `$` es una grafía exclusiva de JS. Permitirlo dejaría que `$tools` pase el worker y falle en todo backend que no sea JS, rompiendo la portabilidad por una ganancia puramente cosmética.

## Consecuencias

Comprado: un solo lugar —el paquete de Service Definition— define qué es un nombre de binding portable, y todo backend aplica el mismo contrato por importación. Una lista de namespaces válida en un backend es válida en todos, de forma verificable, no por coincidencia de qué backend probó el llamador.

Coste: los llamadores existentes del worker que usan un global con `$` ahora fallan la validación de identificadores. Bajo la postura de pre-release esto es una base corregida, no una ruptura de compatibilidad que parchear. Los tests de mal uso del Service Definition del worker ganan casos para `$tools`, miembros de excepción de Python (`args`), dunders (`__dict__`) y un global propiedad de Python (`__dsh_main__`), demostrando que el conjunto compartido se aplica desde el lado del worker.
