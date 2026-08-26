# Agent Note: La búsqueda del rail conserva su expansión cuando el clic de apertura alcanza document

Status: implemented

[English](2026-08-18-rail-search-outside-click-self-dismissal.md) | Español

## Problema

El botón de búsqueda del rail de la barra lateral contraída arma el gesto del rail (`searchOnExpand`), expande el affordance de búsqueda (`searchExpanded`) y solicita la expansión de la barra lateral —diseñado para llevar al usuario a un campo de búsqueda enfocado cuando la columna se deslice abierta. En un navegador real el gesto nunca se completaba: la barra lateral se expandía pero el cuadro de búsqueda permanecía cerrado y sin foco.

El clic iniciador destruye su propio efecto. React despacha el manejador del botón del rail a mitad del burbujeo; el cambio de estado renderiza la cabecera ancha y monta el listener de descarte por clic exterior del WorkspaceBrowser en `document` durante ese mismo despacho. El clic sigue entonces burbujeando y alcanza `document` con el botón del rail, ya desmontado, como objetivo —fuera de `searchRoot`—, de modo que el listener recién montado colapsa de inmediato la búsqueda que estaba abriendo. La prueba del paquete no lo detectó porque `fireEvent.click` sobre el botón no vuelve a burbujear a través de los listeners montados durante el despacho como lo hace un evento real del navegador.

## Decisión

El listener de descarte por clic exterior no se monta mientras el gesto del rail está en vuelo: su efecto retorna antes de tiempo mientras `searchOnExpand` está establecido, y `searchOnExpand` ya termina exactamente cuando el gesto se asienta (el foco aterriza en el campo tras el deslizamiento de la columna). Tras el asentamiento, los clics exteriores descartan la búsqueda como antes. Una prueba de regresión reproduce el orden real del navegador —clic en el rail, giro a ancho y luego el mismo clic llegando a `document`— y exige que la búsqueda permanezca expandida a través de ello y que se descarte con el siguiente clic exterior genuino.

## Alternativas consideradas

**Detener la propagación en el clic del botón del rail.** Suprimir el burbujeo en el iniciador acopla el botón del rail a un listener que no puede ver, y cualquier otra vía de expansión —un futuro atajo de teclado, otra entrada del rail— reintroduciría el bug. El listener es dueño del descarte, así que el listener porta la salvaguarda.

**Aplazar el montaje del listener un frame o un timeout.** Un retardo crudo codifica el síntoma (el clic llega «demasiado pronto») en lugar de la causa (un gesto en vuelo). `searchOnExpand` es ya el estado explícito de gesto en vuelo con el punto final correcto; un límite de frame no es ninguna de las dos cosas.

**Descartar en `pointerdown` en lugar de `click`.** El `pointerdown` del gesto iniciador precede al montaje del listener, así que no puede autodescartarse. Rechazado porque cambia la semántica del descarte para toda interacción —un arrastre o una pulsación con deslizamiento descartarían donde hoy un clic completado no lo hace— para arreglar un problema acotado a un gesto.

## Consecuencias

El gesto de búsqueda del rail funciona de extremo a extremo en la aplicación ensamblada, fijado por un escenario de navegador real de `apps/web`: un clic real viaja a través del rail contraído, el giro a ancho y el burbujeo a nivel de document, y la búsqueda permanece expandida con el foco aterrizando en el campo. Durante la ventana de gesto en vuelo (~300 ms de deslizamiento de columna) un clic exterior no descarta la búsqueda; esa ventana termina en el momento en que aterriza el foco. La prueba de regresión del paquete fija además el momento de la salvaguarda a nivel de unidad.
