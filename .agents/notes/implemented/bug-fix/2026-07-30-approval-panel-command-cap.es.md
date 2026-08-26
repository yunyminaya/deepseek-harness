# Agent Note: La toma de control de aprobación comparte el tope de texto del composer

Status: implemented

English | [中文](2026-07-30-approval-panel-command-cap.zh.md)

## Problema

El panel de aprobación es una toma de control del composer: mientras espera una escalada de sandbox, reemplaza el InputBar en el asiento del composer con la justificación del modelo, el comando emparejado y una fila rehusar/permitir. Ambos textos son salida de modelo sin límite, y la tarjeta no tenía tope de altura. Un comando largo — la forma realista, ya que la escalada ocurre sobre el comando que el sandbox acaba de denegar, y un comando denegado suele ser una escritura inline larga — hacía crecer la tarjeta hasta que la fila de acciones salía del viewport. El usuario podía leer la petición y no responderla: los botones existían, fuera de pantalla, en un footer sticky que ya había usado toda la columna.

El InputBar que el panel reemplaza siempre ha estado capped (14 líneas, luego el textarea hace scroll), así que la toma de control era también el único estado del composer que podía crecer sin límite — la altura del asiento saltaba al electarse y volvía a saltar al responderse.

## Decisión

La justificación y el comando del panel pasan a una región de scroll (`data-approval-scroll`) con tope a la misma altura que el área de borrador del composer; la franja ámbar y la fila de acciones quedan fuera de ella, así que ambos botones están en la tarjeta a cualquier longitud de contenido.

El tope es un valor con dos consumidores, declarado como `--dsh-composer-text-max-height: 336px` en el `.composerSeat` de `ConversationRoot` — el único ancestro compartido de la cadena del composer, ya que el InputBar de fallback y una toma de control electa renderizan como hermanos. El scrollport del borrador de `InputBar` y la región de scroll del panel leen ambos, así que el asiento no puede topear sus dos estados de forma distinta: lo que el diseñador pidió («unifícalo con la altura máxima del input box») es ahora un hecho de la hoja de estilos y no un número repetido en dos archivos. La región es `box-sizing: border-box` así que el tope es su altura exterior, la misma caja que ocupa el área de borrador del composer.

La región es un tab stop (`tabIndex={0}`, con `role="group"`). A diferencia del cuerpo de scroll del composer de preguntas, cuyas filas de opciones son focusable y arrastran al contenedor, esta no sostiene nada más que texto: sin su propio tab stop, un usuario solo-de-teclado podría alcanzar los botones y jamás el final del comando, y aprobar lo que no pudo terminar de leer.

La tarjeta del panel re-liga `--dsh-scrollbar-thumb{,-hover}` al par l2, como toda superficie con scroll sobre un fondo elevado debe ([contrato de scrollbar](../../../../packages/client/ui-theme/src/styles/scrollbar.css)).

## Alternativas consideradas

**Topear la tarjeta entera en vez de la región de texto.** Una declaración, sin reestructurar, y se lee como el literal «misma altura máxima que el input box». Descartado porque la tarjeta sostiene la franja y la fila de acciones: a 336px totales la justificación y el comando quedarían con ~250px, menos espacio que el borrador al que reemplazan, y los números solo coincidirían por casualidad de la altura de la franja. Topear la región de texto hace que ambos asientos topen a la misma altura de texto, que es la propiedad que evita que el footer salte.

**Topear contra el viewport como el composer de preguntas (`min(60vh, 520px)`).** La toma de control hermana ya hace esto, así que es el precedente local. Descartado porque la petición del diseñador era paridad con el InputBar, y las dos tomas de control no tienen la misma forma: el contenido con scroll del composer de preguntas es una lista de opciones que el usuario debe comparar, que quiere todo el viewport que pueda conseguir, mientras que el del panel de aprobación es un comando que el usuario hojea antes de decidir. Un tope relativo al viewport volvería a hacer saltar la altura del asiento al electarse, en la otra dirección.

**Elipsar o truncar el comando.** Sin región de scroll, sin tope, y los botones se quedan. Descartado porque el comando es lo que se aprueba: ocultar su final pide al usuario consentir texto que no puede leer. La truncación además es irrecuperable aquí — el panel es toda la UI de aprobación, así que no hay superficie de «mostrar más» a la que caer.

**Dejar la fila de acciones dentro de la región de scroll y topear la región.** Menos piezas móviles que fijar la fila. Descartado porque reproduce el defecto dentro de la tarjeta: los botones se salen de la región con el scroll, y el usuario tiene que descubrir un scrollbar para alcanzarlos.

## Consecuencias

- Un comando largo hace scroll dentro de la tarjeta y los botones rehusar/permitir permanecen en pantalla. Medido sobre el client construido a 900x1000 y 900x700: la región reporta `scrollHeight` por encima de `clientHeight`, y ambos botones permanecen dentro de la tarjeta y del viewport.
- Elegir la toma de control ya no cambia qué tan alto puede ponerse el asiento del composer, así que la transcripción encima no se re-fluye por cientos de píxeles cuando llega o se resuelve una aprobación.
- El tope de 14 líneas del InputBar ahora se resuelve a través de una custom property heredada de `.composerSeat`, en la caja que hace scroll de su borrador ([un scrollport para ambas capas de texto](2026-07-31-composer-text-layers-share-one-scrollport.es.md) movió la declaración fuera del espejo de auto-crecimiento). Renderizar la barra fuera de ese asiento dejaría caer la declaración (un `var()` sin resolver y sin fallback), así que un futuro host del composer tiene que llevar la propiedad — por eso se declara en el asiento compartido y no en la raíz de la app.
- El comando grabado del escenario es un blob de 200 tokens, mucho más largo de lo que un round trip necesita. Ese costo es deliberado: el tope es infalsificable sin contenido que lo supere, y el modelo comprime cualquier payload regular (la primera grabación convirtió «alpha 400 veces» en `printf 'alpha %.0s' {1..400}`, un comando de una línea que no prueba nada).

## Verificación

`apps/web/tests/approval-composer.e2e.ts` conduce la composición real: una sesión de solo-lectura, una escritura denegada, el reintento de escalada del modelo, y la respuesta clickeada a través del panel. La aserción de geometría corre sobre el panel vivo a dos alturas de viewport y está guardada contra sostenerse vacuamente — la región debe estar realmente haciendo scroll, y el tope medido debe igualar el propio del composer, que el test lee del scrollport de borrador vivo antes de enviar, en vez de hardcodear el valor en px.

Confirmado en ambas direcciones contra el client construido. Con el tope revertido, la región reporta `scrolls: false` y crece a la altura completa del comando (1798px para el blob grabado a 900x1000, contra 336px con tope); a 900x700 la tarjeta mide 680px contra un viewport de 700px y el fondo de la fila de acciones aterriza en y=749 — bajo el pliegue, exactamente el reporte del diseñador. Con el tope restaurado el escenario pasa en replay.

Reproducir los botones fuera de pantalla necesita una tarjeta más alta que el scrollport, no meramente una tarjeta alta. El asiento del composer es `position: sticky; bottom: 0`, así que mientras la tarjeta aún cabe permanece pegada al fondo del viewport y los botones siguen visibles — a 900x1000 la tarjeta sin tope se comió toda la transcripción y aun así mantuvo su fila de acciones en pantalla. Solo cuando la tarjeta supera al scrollport sticky deja de poder sostener el borde inferior, y la fila se va abajo.

El bloque de geometría y el golden son solo-replay, así que el modo record alcanza la escritura del fixture en vez de abortar en el layout.

El escenario mantiene exactamente un golden — el panel en espera — y aserta el estado respondido sobre el mundo (el resultado decidido, el archivo que el comando escalado escribió, `DONE`, el panel desaparecido, el composer re-habilitado). Un golden de transcripción-respondida no puede sostenerse: el primer intento denegado renderiza la negativa propia del SO, y ese texto es platform-specific (`bash: notes.txt: Operation not permitted` en macOS contra `bash: line 1: notes.txt: Read-only file system` en Linux). Cualquier escenario cuya transcripción contenga un comando denegado por sandbox hereda eso, así que la denegación pertenece a las aserciones, jamás a un golden.

El panel shipea como bundle de client-module: `pnpm run build:web` solo no recoge un cambio en `ApprovalPanel.module.css` ni un hook `data-` nuevo en `ApprovalPanel.tsx` — el build del paquete debe correr primero, o el lane del browser aserta contra un client más viejo que el árbol.
