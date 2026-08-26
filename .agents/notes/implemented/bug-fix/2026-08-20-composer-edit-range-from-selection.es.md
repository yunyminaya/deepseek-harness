# Agent Note: Las ediciones del composer portan el rango sobre el que se aplicaron

Status: implemented

[English](2026-08-20-composer-edit-range-from-selection.md) | Español

## Problema

La máquina de entrada mantiene alineadas sus ocurrencias de referencia reconciliándolas contra un único rango de edición: las entradas anteriores al rango se desplazan, las posteriores se mantienen, y una entrada que el rango intersecta pierde su identidad estructurada y queda como texto de borrador ordinario. Esa última regla es el significado deliberado de editar dentro de una referencia.

La escritura ordinaria no aportaba ningún rango. El evento change de un textarea controlado lleva solo la cadena resultante, así que la máquina recuperaba el rango escaneando los dos borradores en busca de un prefijo y un sufijo comunes. Esa recuperación es ambigua siempre que el texto insertado repite el texto contra el que aterriza, y el escaneo voraz resuelve siempre la ambigüedad del mismo modo: desliza la edición tan tarde como los caracteres permitan.

Una referencia se renderiza como `@` seguido de su etiqueta, así que teclear `@` inmediatamente antes de una produce exactamente esa colisión. El usuario inserta en el propio offset de la referencia; el escaneo informa de una inserción un carácter más tarde, dentro de la referencia; reconcile aplica la regla de intersección y descarta la ocurrencia. Borrar un carácter delante de una referencia así se desliza del mismo modo.

El borrador sigue leyéndose entonces correctamente a la vista mientras no porta ninguna referencia estructurada, y el envío toma la vía sin ocurrencias que manda el borrador verbatim. El host recibe la etiqueta orientada al humano en lugar de la forma de modelo del dueño y no resuelve nada. La salvaguarda de serialización que existe para impedir exactamente esta degradación nunca se ejecuta, porque solo se dispara cuando una ocurrencia sobrevive para ser serializada.

Esto se volvió alcanzable cuando las referencias [se convirtieron en texto inline literal](../feature/2026-07-27-web-file-and-session-references.es.md). Una referencia ocupaba antes un único `U+FFFC`, un carácter que ninguna pulsación produce, así que el escaneo no tenía con qué colisionar.

## Decisión

`InputBar` registra la selección del textarea y el `inputType` durante `beforeinput` y pasa el rango resultante a `setDraft`, que la máquina ya acepta y prefiere sobre su propio escaneo. Un textarea no expone la edición de ninguna otra manera —`getTargetRanges()` está vacío para los controles de formulario.

Una edición que reemplaza una selección informa de esa selección, y es el rango directamente; la longitud insertada es lo que el borrador haya crecido una vez contabilizado el rango reemplazado. Un borrado con caret no reemplaza nada e informa del caret desnudo, así que su rango procede de la dirección que nombra `inputType` y del número de caracteres que el borrador perdió realmente. La cuenta se mide en lugar de suponerse uno, porque un único gesto de caret elimina igual de fácilmente un grafema multiunidad, una palabra o una línea. Chromium, WebKit y Firefox informan todos del caret colapsado para `deleteContentBackward` y `deleteContentForward`, y los tres derivan el mismo rango de él.

Solo se registran las familias `insert` y `delete`. Una reproducción de historial informa de dondequiera que esté el caret, lo que sobreviviría a todas las comprobaciones mientras nombra el tramo equivocado; ignorarla deja esa vía en el escaneo.

El registro se consume una vez y se limpia. Un registro cuya longitud de borrador discrepa del borrador que informa el change, una selección más allá del borrador, una edición que encoge sobre una selección que el caret no puede explicar, o un borrado sin dirección producen todos ausencia de rango, y la máquina cae a su escaneo. El pegado y los gestos de Backspace y Delete en el límite ya aportaban sus propios rangos y no se tocan.

## Pruebas

Las pruebas de componente cubren el carácter disparador tecleado delante de una referencia, un Backspace con caret, un Delete con caret, un borrado de palabra con caret y un borrado sobre una selección, verificando en cada caso que la ocurrencia sobrevive en el offset desplazado. Los casos de caret fallan contra el rango recuperado por escaneo, y el caso de palabra falla contra un paso fijo de un carácter.

Un escenario de navegador ensamblado acciona los mismos gestos como pulsaciones reales contra la composición publicada, que es el único lugar donde puede observarse el rango que un motor informa para ellos; su golden proyecta los segmentos del backdrop, porque la capa de decoración es aria-hidden y el árbol de accesibilidad no puede ver el chip. Una composición real accionada a través de la vía IME del navegador informa del segmento en composición como selección en cada estado intermedio, y la referencia sobrevive a cada uno.

## Alternativas consideradas

**Desambiguar el escaneo con el caret posterior a la edición.** El caret fija cuál de las lecturas textualmente equivalentes ocurrió, y el evento change ya lo porta. Rechazado porque conserva una reconstrucción donde hay disponible un hecho exacto, y no puede separar en absoluto las mitades borrada e insertada de una selección reemplazada.

**Dar a las referencias un marcador inicial que ninguna pulsación produzca.** Un carácter de uso privado en lugar del `@` literal elimina la colisión a nivel de representación, y la lista de rechazo para texto pegado ya nombra ese rango. Rechazado porque vuelve a añadir un carácter que toda serialización, selección y vía de accesibilidad tiene que eliminar, para comprar lo que un rango exacto compra directamente, y dejaría a la escritura ordinaria reconstruyendo su rango por cualquier otra razón.

**Ampliar reconcile para conservar una ocurrencia cuando el rango solo toca su borde.** Rechazado porque el offset mal atribuido aterriza estrictamente dentro de la referencia, no en su límite, de modo que el cambio de regla no alcanzaría este defecto y volvería más vaga la regla de intersección.

## Consecuencias

Toda edición nativa del textarea que llega a `onChange` nombra ahora el rango sobre el que se aplicó, de modo que los offsets de las ocurrencias siguen la edición que ocurrió realmente en lugar de una meramente consistente con los caracteres resultantes. Las escrituras de borrador que se originan en la fachada y no en el DOM —`insertText` y los empalmes de token de comando entre ellas— siguen sin portar rango y conservan el escaneo, al igual que los respaldos anteriores.

El composer depende ahora de que `beforeinput` preceda a cada cambio de valor, y de que `inputType` nombre la dirección de un borrado con caret. Cualquier vía de edición futura que mute el valor sin ninguna de las dos vuelve silenciosamente al escaneo en lugar de romper, lo que conserva el modo de fallo del comportamiento antiguo en lugar de un rango equivocado.
