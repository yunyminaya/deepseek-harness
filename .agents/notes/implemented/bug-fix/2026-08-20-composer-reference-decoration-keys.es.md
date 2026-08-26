# Agent Note: Las decoraciones de referencia del composer se clavean por ordinal de orden en el borrador

Status: implemented

[English](2026-08-20-composer-reference-decoration-keys.md) | Español

## Problema

El backdrop del composer renderiza el borrador como un array de segmentos: cadenas planas, una marca de token de reclamo inicial, un elemento por referencia estructurada y una marca por rango de referencia de texto plano. React reconcilia ese array por clave.

Las referencias estructuradas portan una identidad —la tabla de ocurrencias acuña un `occurrenceId` que sobrevive a toda edición—, así que sus chips se clavean por ella. Los rangos de referencia de texto plano no tienen esa identidad: `scanTextRefs` los vuelve a derivar del borrador en cada render, y nada fuera de ese escaneo recuerda un rango entre dos pulsaciones.

Clavear esos rangos por su offset en el borrador hacía que la clave cambiara cada vez que el texto anterior cambiaba de longitud. React trataba entonces el rango como un elemento distinto, desmontaba la marca con sus spans anidados y su glifo inline, y montaba un reemplazo. Cada carácter tecleado o borrado delante de una referencia reconstruía toda referencia posterior al caret, y el trabajo crecía con el número de referencias. Los [rangos de sintaxis de directorio](../feature/2026-07-27-web-file-and-session-references.es.md) volvieron rutinaria esa vía: coinciden por sintaxis `@path/` sin léxico, y cada uno renderiza un icono.

## Decisión

Una marca de referencia de texto plano se clavea por su índice en la lista `textRefs` ordenada por offset, calculado donde se ensambla la lista de límites para que un límite omitido no pueda desplazarlo. El escaneo ya devuelve los rangos en orden de borrador, así que el ordinal nombra la ranura de render que ocupa un rango, que es la única identidad que tiene un rango derivado del escaneo.

Los chips estructurados conservan `occurrenceId`. Las dos estrategias de clave difieren porque los dos tipos de rango difieren en identidad, no por descuido: un rango que posee la tabla de ocurrencias conserva su nodo a través del reordenamiento, y un rango que solo conoce un escaneo conserva su nodo a través de los desplazamientos de offset.

Un rango que deja de coincidir con el escaneo sigue perdiendo su decoración, porque desaparece de `textRefs` y el ordinal que ocupaba ya no existe.

## Pruebas

Una prueba de componente mantiene el elemento de la marca y su glifo, teclea un carácter delante del rango y verifica que los mismos nodos siguen montados; luego edita el token fuera de la forma de coincidencia y verifica que la decoración ha desaparecido. La prueba falla contra una clave derivada del offset.

## Alternativas consideradas

**Clavear por el texto del rango.** Rechazado: las referencias duplicadas colisionan en una sola clave, y editar dentro de un rango cambia su clave, lo que reintroduce el remontaje que esto arregla.

**Dar a los rangos derivados del escaneo una tabla de identidad.** Rechazado: añade estado mutable cuyo único consumidor es una clave de render, y el escaneo tendría que hacer un diff contra el borrador anterior para mantenerla. Que una edición que rompe una coincidencia simplemente descarte el rango en el siguiente escaneo es lo que mantiene `scanTextRefs` como derivación pura.

**Quitar las claves y dejar que React coincida por posición.** Rechazado: React exige claves en los elementos dentro de un array, y los segmentos de cadena plana entre ellos ya coinciden por índice, así que un elemento sin clave avisa sin cambiar el resultado.

## Consecuencias

Teclear delante de una referencia actualiza solo los nodos de texto; la marca y su icono permanecen montados. El trabajo DOM por pulsación del backdrop ya no escala con el número de referencias en el borrador.

Como la clave nombra una posición, insertar una referencia delante de las existentes reutiliza los nodos anteriores con contenido nuevo en lugar de recrearlos. Eso es correcto para estas marcas, que no mantienen estado de foco, selección ni animación, y es la condición que cualquier decoración futura de esta capa cumple antes de clavearse por ordinal.
