# Agent Note: La columna de conversación hace scroll en un eje

Status: implemented

English | [中文](2026-08-04-conversation-column-one-axis-scroll.zh.md)

## Problema

Estrechar la columna central — por la ventana o por el arrastre de la barra lateral — ponía un scrollbar horizontal bajo toda la columna de conversación en el hero. El elemento que sangra es la elipse decorativa de fondo del hero: `.heroGlow` se dimensiona a `1051/776` de la caja del hero para que su blur escale en userSpace con la tarjeta de input, lo que significa que alcanza más allá de la columna siempre que la columna es más estrecha que el glow.

Ese sangrado es por construcción y se queda. Lo que lo hacía visible al usuario es el contenedor de scroll en que se sienta. `[data-conversation-scroll]` declaraba `overflow-y: auto` y dejaba el otro eje en su `visible` inicial, y una caja que scrollea en un eje computa `visible` a `auto` en el otro. Toda columna más estrecha que el glow ofrecía por tanto un rango real de scroll horizontal — medido en 24–95px a lo largo de los anchos que un portátil de verdad produce.

## Decisión

`.scrollBody` declara `overflow-x: hidden`. La columna enuncia que es un scroller de un eje en vez de dejar que el segundo eje se derive.

El recorte no cambia. `overflow-y: auto` ya había hecho de la caja un contenedor de scroll que recorta ambos ejes, así que la declaración retira solo el scrollbar y el gesto del usuario; el glow conserva su sangrado, su radio de blur y el mismo extent pintado, y la columna conserva su scroll vertical. Nada en la cadena del composer se mueve.

## Alternativas consideradas

**Dimensionar el glow a la columna.** Descartado. El ancho del glow es lo que escala su blur `stdDeviation="50"` con la tarjeta de input (figma 313:14109); restringirlo apretaría el blur al estrecharse la columna, una regresión visual para arreglar un scrollbar.

**Envolver el glow en una caja de recorte.** Descartado. Añade una caja cuyo único trabajo es deshacer un overflow que la columna ya recorta, y deja el `overflow-x: auto` derivado en su sitio para el siguiente elemento que sangre — la transcripción está llena de candidatos.

**Confiar en el `.centerCol { overflow: hidden }` del frame.** No puede ayudar. Ese clip está fuera del contenedor de scroll, así que esconde el vuelo del glow en el borde de la columna mientras el contenedor dentro de él sigue scrolleando para alcanzarlo. La barra reportada era la de ese contenedor.

**Asertar `scrollWidth === clientWidth` en el test.** Descartado como señal, porque no distingue los estados: `hidden` recorta el sangrado en vez de re-fluirlo, así que el rango de scroll lee igual a ambos lados del fix. Solo rehusar un gesto de usuario difiere, que es lo que el escenario mide.

## Testing

[apps/web/tests/conversation-column-overflow.e2e.ts](../../../../apps/web/tests/conversation-column-overflow.e2e.ts) barre anchos de viewport abrazando el glow y, en cada parada, hace rueda horizontal sobre la columna y lee `scrollLeft`. El golden commiteado registra la relación por parada; la parada más ancha es el control donde el glow no sangra en absoluto.

Dos guards mantienen honesto el escenario. El guard de vacuidad aserta que el glow sigue alcanzando más allá de la columna en las paradas estrechas, así que la afirmación no puede pasar porque el síntoma haya desaparecido por una razón no relacionada. El control de mutación fuerza `overflow-x: auto` de vuelta en la página y muestra el mismo gesto, al mismo timing, llevando la columna a su frontera positiva de scroll; el test mide esa frontera directamente porque un gutter de scrollbar estable puede dejar algo de overflow en el lado negativo del origen de scroll. Sin el control, un `scrollLeft` de 0 podría igualmente significar que la rueda jamás llegó.

## Consecuencias

La columna de conversación ya no ofrece scrollbar horizontal a ningún ancho, y el sangrado decorativo en la cadena del composer queda recortado en vez de expuesto como rango de scroll. El costo es que el contenido genuinamente ancho bajo esta columna se recorta en vez de ser alcanzable con scroll: toda superficie así posee su propio scroller, como ya hacen el bloque de código markdown y la tabla de trayectoria.
