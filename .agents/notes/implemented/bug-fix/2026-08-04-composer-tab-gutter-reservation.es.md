# Agent Note: La columna de conversación reserva un gutter de scrollbar para cada vista

Status: implemented

English | [中文](2026-08-04-composer-tab-gutter-reservation.zh.md)

## Problema

El asiento del composer es un nodo en un lugar del árbol, y se trazaba contra un borde distinto según la pestaña de vista mostrada.

En Chat es hijo sticky del scroller de la columna (`[data-conversation-scroll]`), así que monta la content box de ese scroller — la caja que un scrollbar consumidor de espacio acorta en el ancho de la barra. Una vista que declara `data-conversation-composer-overlay`, como hace Trajectory, mueve el scroll de la columna a la vista misma: la rama keyed en ese atributo dejaba el scroller `overflow: hidden` y posicionaba el asiento absolutamente, contra la padding box, que ningún scrollbar reduce.

Así que mientras la transcripción desbordaba — el estado ordinario de cualquier sesión con historial — las dos pestañas discrepaban exactamente en el ancho de la barra. La tarjeta de input está centrada, así que cambiar de pestaña la movía 4px de lado en una barra de 8px, y su holgura derecha cambiaba los 8 completos. El mismo desplazamiento aparecía dentro de Chat solo, en el momento en que una transcripción creciente empezaba a scrollear, y otra vez entre la fase hero y el primer turno con scroll.

## Decisión

`.scrollBody` declara `scrollbar-gutter: stable` para el estado Chat, y la rama overlay la sobreescribe con `scrollbar-gutter: auto` permaneciendo contenedor de scroll en ambos ejes — `overflow-x: hidden; overflow-y: auto`. La reserva es solo de Chat: mantiene la content box del asiento al mismo ancho desborde o no la transcripción, así que la tarjeta jamás salta cuando una transcripción creciente empieza a scrollear, ni entre la fase hero y el primer turno con scroll. La rama overlay no reserva nada — la vista posee sus propios scrollers, así que un gutter allí solo estrecharía el contenido de la vista — y su asiento compensa la barra en su lugar ([la compensación de ancho del asiento](2026-08-12-composer-overlay-seat-width-compensation.es.md)).

`stable` y no `auto` porque `auto` reserva solo mientras la caja efectivamente desborda, y la diferencia entre desbordar y no es precisamente la diferencia entre las dos fases de Chat — un gutter `auto` enunciaría el bug en vez de arreglarlo.

La reserva vive en una caja `overflow-y: auto`, y esa forma es load-bearing: WebKit aplica `scrollbar-gutter` a una caja `overflow-y: auto` y la ignora en una hidden — medido en las propias capas del composer de esta app y registrado en [la nota del scrollport del composer](2026-07-31-composer-text-layers-share-one-scrollport.es.md) — así que una reserva en una caja hidden sostendría en Chromium y silenciosamente no en Safari. La rama overlay conserva también su forma `overflow-y: auto`, como caja de recorte de la que nada se sale con scroll: un scroller de un solo eje computa el otro eje a `auto`, así que el eje horizontal se declara `hidden` en vez de dejarse computar, y de otro modo haría crecer un scrollbar horizontal propio la primera vez que el contenido de una vista superara la columna.

La reserva vale lo que cuesta solo porque la barra aquí toma espacio de layout en absoluto, que no es el comportamiento por defecto del browser sino el de este client: el `::-webkit-scrollbar` lleva un ancho en la hoja de ui-theme ([scrollbars tematizados](2026-07-28-themed-scrollbars-and-reserved-gutter.es.md)), y la lista de sesiones de la barra lateral ya reserva su propio gutter por la misma razón.

## Alternativas consideradas

**Insertar (inset) el asiento overlay el ancho de la barra.** La lectura estrecha del bug — los dos estados difieren en 8px, así que restar 8px a uno. Descartado porque el número es del engine, no nuestro: la ruta WebKit dibuja la barra de 8px de la hoja, la ruta Firefox dibuja lo que `scrollbar-width: thin` resuelva, y un inset hardcodeado alinearía los dos estados en Chromium desviándose en todo lo demás. El gutter le pide al engine reservar el ancho de su propia barra, sea cual sea.

**Conservar `overflow: hidden` y añadir solo `scrollbar-gutter: stable`.** La versión de una línea. Arregla el síntoma visible en el engine que corre el lane del browser, y lo deja en Safari, sin que ningún test falle en ninguna parte — el modo de fallo que la segunda mitad del cambio existe para prevenir.

**Sacar el asiento del composer del scroller también en Chat, haciendo de la geometría overlay la única geometría.** Esto borra la diferencia en su raíz en vez de reconciliarla, y cede una propiedad deliberada: el asiento sticky vive dentro del flujo de scroll, así que una rueda sobre el composer mueve la transcripción ([composer sticky](2026-07-29-sticky-composer-conversation-scroll.es.md)), y la máscara de desvanecimiento encima la pinta el fondo del propio asiento. Ambos son comportamiento con dueño y cobertura propia; reconstruirlos para quitar 8px de asimetría es el cambio mayor, no el menor.

**Acolchar (pad) la columna el ancho de la barra en vez de reservar un gutter.** El padding aplica haya o no barra presente, así que cuesta el ancho incondicionalmente en cada estado, y fija en la hoja de estilos un valor que el engine elige en tiempo de layout. Descartado por la misma razón por la que la lista de la barra lateral lo rechazó.

## Consecuencias

- La columna de contenido de Chat queda permanentemente 8px más estrecha — en la fase hero y con la transcripción corta también, donde no se dibuja barra. Ese es el trade: una posición de tarjeta a cada altura de contenido, en vez de la columna más ancha posible.
- La tarjeta sostiene una posición a través de tres transiciones, por dos mecanismos: la reserva mantiene el asiento de Chat a un ancho a través de sus propias fases (transcripción corta ↔ con scroll, hero ↔ primer turno scrolleante), y la compensación del asiento overlay la iguala en la transición Chat ↔ Trajectory ([la compensación de ancho del asiento](2026-08-12-composer-overlay-seat-width-compensation.es.md)).
- El estado overlay es ahora un contenedor de scroll. Nada en él puede desbordar hoy; una vista futura que dejara su contenido superar la columna haría scroll de esta caja en vez de recortarla, y necesitaría su propio clip como la vista Trajectory ya tiene uno.
- El golden commiteado registra la banda reservada, así que un cambio al ancho `::-webkit-scrollbar` de la hoja — el valor que decide qué tan ancha es la reserva — llega como diff revisable en este escenario igual que en el de la barra lateral.

## Testing

`apps/web/tests/composer-tab-geometry.e2e.ts` mide el rectángulo de la tarjeta de input en ambas pestañas, a un viewport donde la tarjeta se sienta en su tope de ancho y a uno donde se encoge con la columna, y aserta que los dos rectángulos son el mismo rectángulo. Solo un engine real reporta esto: jsdom da a todo elemento una caja de tamaño cero y sin scrollbar, así que una spec unitaria podría asertar que las declaraciones existen pero no que los dos estados aterrizan en el mismo sitio. Por la misma razón no la acompaña ninguna spec de texto-CSS — restataría las declaraciones sin añadir un hecho que el lane del browser ya no establezca.

El escenario lanza chromium sin el `--hide-scrollbars` por defecto de Playwright, que es load-bearing: bajo ese argumento una barra no consume ancho de layout, así que las pestañas coinciden con y sin la compensación y toda comparación del archivo se sostiene vacuamente. Medido, ambas bandas se sientan en 0 bajo el argumento y en 8 y 0 con él caído.

La cascada sin compensar se aplica entonces en la página — la compensación `right` del asiento overlay caída a 0 vía `!important`, la reserva de Chat intacta — y las mismas dos pestañas medidas a través de ella, que es lo que separa una tarjeta que no se mueve de un cambio de pestaña que jamás alcanzó el layout. Reproduce el síntoma reportado como número: 4px en cada borde, la mitad de la banda de 8px. El golden registra ese control junto al estado fijado, así que el fixture lleva la diferencia que el cambio elimina y no solo su ausencia.
