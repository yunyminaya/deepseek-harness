# Agent Note: Cabecera fija y composer sticky dentro del scrollport del transcript

[English](2026-07-29-sticky-composer-conversation-scroll.md) | [中文](2026-07-29-sticky-composer-conversation-scroll.zh.md) | Español

Estado: implementado

## Problema

La columna de conversación activa dividía el desplazamiento: la vista de chat (y la de trayectoria) era dueña de `overflow-y: auto`, mientras que la pila del composer quedaba como hermana por debajo de ese scrollport. Un gesto de rueda sobre la línea de estadísticas o la entrada golpeaba por tanto una región sin desplazamiento y no hacía nada — el transcript (transcripción) solo se movía cuando el puntero estaba sobre la lista de mensajes. Los borradores largos lo empeoraban: el textarea es en sí mismo un scrollport, así que la rueda sobre el composer podía quedar atrapada allí. La cabecera de sesión debe ocupar la parte superior de la columna como chrome ordinario (no `position: sticky` dentro del scrollport), mientras que el composer debe quedarse pegado al fondo del mismo scrollport que el transcript para que la rueda sobre el pie mueva el flujo.

## Decisión

`ConversationRoot` es dueño siempre de un único cuerpo `data-conversation-scroll`, con el outlet de vista estricto `conversation.session` antes de un `data-composer-seat` alrededor de toda la salida de la cadena `'conversation.composer'` (respaldo + hermanos overlay elegidos de `overlay: true`). El outlet estricto separado `conversation.session.header` sigue siendo chrome de columna `flex: none` por encima de ese scrollport y se oculta mientras la sesión está en blanco. Este árbol padre fijo mantiene montados el cuerpo de scroll y el asiento del composer desde ninguna sesión, pasando por el Hero en blanco, hasta la conversación activa. El CSS activo fija ese asiento con `position: sticky; bottom: 0` para que las tomas de control de Question/Approval sigan visibles cuando el usuario no está anclado al fondo; el CSS del Hero centra la pila de respaldo dentro del cuerpo de scroll. ChatView y Trajectory/Waterfall conservan un scroller local solo cuando se montan fuera de ese host (pruebas unitarias); bajo el host establecen `overflow: visible` y resuelven el seguimiento del fondo y el anclaje de prepend mediante `closest('[data-conversation-scroll]')`.

Las estadísticas de sesión viven en `'conversation.composer.dock'` (por encima de `'conversation.input.dock'`). El textarea de InputBar, cuando está dentro del host, encadena `wheel` con `{ passive: false }`: mientras el textarea con tope pueda seguir desplazándose en esa dirección conserva el gesto nativo; solo en su propio borde hace `preventDefault` y aplica `deltaY` al host.

El prepend del historial de chat sigue la intención del lector mediante identidades estables de nodo/llamada renderizadas en lugar de deltas de altura de todo el scrollport. `ChatView` registra la primera `data-chat-anchor-key` visible y su parte superior relativa al scrollport cuando comienza la paginación, vuelve a seleccionar el ancla estable actualmente visible tras cada desplazamiento del lector mientras la petición está en curso, y compensa con el delta del rectángulo posterior al prepend de esa fila. Llegar al fondo o añadir el propio mensaje del lector cancela el ancla de paginación, así que una página tardía no puede apartar la vista del contenido más nuevo. El seguimiento del fondo es estado almacenado y no geometría bruta de scroll; cómo se reconoce la entrada del lector — la desviación agnóstica de dispositivo respecto al ledger de top observado de la última `scrollTop` entregada o escrita — es competencia de la [nota de atribución de scroll del lector](2026-08-06-reader-scroll-attribution-observed-top-ledger.es.md). El único `ResizeObserver` de ChatView sigue el streaming, la divulgación de herramientas y el redimensionado del borrador solo mientras la propiedad del fondo sigue anclada, sin una segunda escritura de scroll por chunk.

## Alternativas consideradas

**Cabecera sticky y composer sticky dentro de un único scrollport de columna.** Rechazada para la cabecera: debe ocupar la parte superior como chrome de diseño fijo, no participar en la capa sticky del scrollport.

**Composer fijo flex-none por debajo del scrollport con reenvío de rueda.** Rechazada: el producto exige que el composer se quede pegado dentro del scrollport del transcript para que el pie sea parte de esa superficie de hit-testing de scroll, no una hermana que solo reenvía deltas.

**Portalizar el composer al scroller de ChatView.** Rechazada: el composer se comparte entre las pestañas de vista; su destino es el scrollport propiedad de la raíz en el shell residente.

**Mantener StatsLine dentro de ChatView bajo la columna de mensajes.** Rechazada: fuera del composer sticky se desplazaría mientras la entrada quedaba anclada.

**Modelar cada fuente de entrada de scroll del navegador.** Rechazada para esta corrección acotada: la ruta de escritorio reproducida usa entrada de rueda/trackpad. El desplazamiento por puntero/táctil, el arrastre de la barra de desplazamiento nativa, el desplazamiento por teclado, la navegación por foco y la propiedad de overflow anidada quedaron fuera del modelo de fuentes de entrada en lugar de añadir una máquina de estados de entrada general. La [nota de atribución de scroll del lector](2026-08-06-reader-scroll-attribution-observed-top-ledger.es.md) cerró después este aplazamiento generalizando la atribución mediante el ledger de top observado, todavía sin máquina de estados de entrada.

## Consecuencias

La rueda sobre el pie desplaza el transcript; el diseño visible es una cabecera fija, un transcript con desplazamiento y un composer sticky al fondo. Las estadísticas aparecen en cada pestaña de vista activa. Los scrollers de vista anidados bajo el host se suprimen para que las cabeceras Turn sticky de Trajectory se queden pegadas al host de columna. El historial concurrente, el streaming, la expansión de herramientas y el reflow del composer preservan las decisiones de scroll del lector, incluida la entrega compositor-primero de Chromium y el recorte de encogimiento al finalizar el stream. La propiedad de seguimiento se extiende a toda entrada del lector según la [nota de atribución de scroll del lector](2026-08-06-reader-scroll-attribution-observed-top-ledger.es.md). Ni la transición de ninguna sesión al Hero en blanco ni la del Hero a activa pierden el mismo nodo DOM del textarea ni el borrador de InputHub.
