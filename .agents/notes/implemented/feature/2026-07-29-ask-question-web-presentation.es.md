# Agent Note: Presentación web de ask-question

Status: implemented

[English](2026-07-29-ask-question-web-presentation.md) | Español

## Problema

La GUI Web ya podía recoger respuestas a través de la toma de control del compositor `QuestionComposer`, pero el transcript que la rodeaba estaba mal en tres aspectos. Una pregunta pendiente se renderizaba dos veces: una como toma de control del compositor y otra como el placeholder de solo lectura `PendingCard` que es anterior a la toma de control. Una llamada resuelta de `ask_user_question` se renderizaba como la fila genérica «Tool call» volcando el JSON crudo de los args, así que los dos veredictos del compositor — el usuario descartando todo el conjunto (`ASK_CANCELLED`) y una interrupción de turno que aterrizaba mientras la pregunta estaba pendiente (`ASK_ABORTED`) — se leían ambos como fallos anónimos de punto rojo. Y el chrome propio del compositor (paginador, botones, placeholders, feedback de validación) estaba codificado en chino mientras el resto del cliente es bilingüe a través de `dsh-client-locale`.

Aparte, los visuales del compositor habían derivado del diseño actual: una entrada de respuesta personalizada expandible para abrir, sin affordance de multi-selección más allá de una marca final, paginación montada en la cabecera y una convención de sufijo de título `（可多选）` parseada del texto del modelo.

## Decisión

Una pregunta pendiente es dueña de exactamente dos superficies: la toma de control del compositor recoge las respuestas, y una fila de toolview dedicada `ask_user_question` en el transcript nombra el resultado de la interacción. La fila se registra en el hueco con clave `tool.call.toolview` exactamente como `todo_write` y compone el `ToolRow` compartido (chrome, sweep de ejecución, expansión principal). Su resumen es el veredicto de la interacción en lugar de los args: `waiting` mientras se ejecuta, `N/M answered` del JSON de resultado una vez resuelto (una respuesta omitida — `selected` vacío, sin `custom` — queda fuera del recuento), `cancelled` para `ASK_CANCELLED` e `interrupted` con la semántica compartida de parada ámbar para `ASK_ABORTED`. Los resultados malformados o truncados caen al resumen genérico. `PendingCard` se estrechó a `PendingWait<'approval'>` y `ChatView` filtró la lista de pendientes a las esperas de aprobación, dejando la tarjeta placeholder solo para las aprobaciones; la toma de control de aprobación del compositor ([permiso y aprobación web](2026-07-23-web-permission-and-approval.es.md)) la ha eliminado desde entonces por completo.

El rediseño del compositor mueve la paginación al pie junto a las acciones, renderiza las opciones de multi-selección con casillas de verificación explícitas, conserva las filas numeradas de selección única y reemplaza la entrada personalizada expandible para abrir por una fila de entrada personalizada siempre visible (textarea para preguntas sin opciones). Se elimina la convención de sufijo de multi-selección de `parseQuestionTitle`; `multi_select` ya es metadatos estructurados, así que el título se renderiza verbatim.

El chrome del compositor se vuelve bilingüe: el plugin registra diccionarios zh/en bajo el namespace `question` de `dsh-client-locale` y entrega a la entrada un traductor ligado al namespace más la instantánea de locale como fuente del compartimento de hooks a través de la cara de inyección del slot, de modo que un cambio de locale re-renderiza un compositor montado. El feedback de validación se almacena como clave de diccionario y se vuelve a traducir al cambiar; los mensajes de fallo del carrier y todo el texto de pregunta/opción escrito por el modelo se renderizan verbatim.

Dos arreglos adyacentes viajan de paso. Todos los iconos principales de toolview genéricos (y el chevron de hover) heredan ahora el único color de etiqueta terciario — se eliminan el override secundario de la variante others y la regla separada de color del chevron, dejando solo el acento business-primary intencional de cordis. Y el bundler de dev-watch del cliente registra cada módulo CSS con `addWatchFile`, porque la indirección de módulo virtual ocultaba antes del watcher las ediciones solo CSS.

## Alternativas consideradas

**Seguir renderizando las preguntas a través de `PendingCard`.** Rechazada: la tarjeta era un placeholder de solo lectura de antes de que existiera la toma de control, así que una pregunta pendiente mostraba el mismo contenido dos veces con una copia no respondible. La fila de toolview más la toma de control cubren tanto el registro del transcript como la superficie de recogida.

**Mostrar las preguntas o respuestas en línea en la fila del transcript.** Rechazada: la toma de control del compositor es dueña del render de preguntas y de la recogida de respuestas, y la convención de fila (`todo_write`) es una línea con los detalles en el panel. La fila reporta por tanto solo el resultado, reflejando cómo la fila de todo reporta recuentos mientras el panel es dueño de la lista.

**Renderizar `ASK_CANCELLED`/`ASK_ABORTED` a través de la forma de error genérica.** Rechazada: el descarte es la propia acción deliberada del usuario y una interrupción es el gesto de parada compartido; ambos son resultados esperados, no fallos de herramienta. Nombrar el veredicto (y conservar la semántica ámbar de parada para el abort) coincide con cómo se leen las llamadas de herramienta interrumpidas en otros sitios.

**Traducir ahora los veredictos de la fila.** Diferido por decisión explícita de producto: las cadenas `waiting`/`answered`/`cancelled`/`interrupted` de la fila siguen en inglés para este cambio; la i18n del chrome del compositor aterrizó porque su copia solo en chino ya era incorrecta para el locale en.

**Conservar la convención de multi-selección por sufijo de título.** Rechazada: `multi_select` es metadatos de solicitud estructurados y el affordance de casillas lleva ahora la señal, así que parsear `（可多选）` del texto del modelo era un canal duplicado frágil.

## Consecuencias

`ask_user_question` y `todo_write` demuestran ahora el patrón de toolview previsto: componer `ToolRow`, resumir desde los args de llamada o el JSON de resultado con fallbacks comprobados por forma, y registrarse a través del slot con clave. El `todo-row.module.css` a medida ya no existe.

Las cadenas de veredicto de la fila son la única superficie restante con inglés codificado del flujo de preguntas; localizarlas es seguimiento diferido. La toma de control de aprobación del compositor se publicó ([permiso y aprobación web](2026-07-23-web-permission-and-approval.es.md), con tope de altura según la [nota del panel de aprobación](../bug-fix/2026-07-30-approval-panel-command-cap.es.md)), y `PendingCard` ya no existe.

`ui-user-questions` gana una dependencia de `dsh-client-locale` y una cara de inyección donde antes no tenía ninguna; su contrato (`QuestionComposerInjected`) vive con el consumidor en `contract/slots.ts`.

## Verificación

Las pruebas de `ui-conversation` fijan la matriz waiting/answered/skipped/cancelled/interrupted/fallback de la fila, el filtro de pendientes solo aprobación y el registro del slot; las pruebas de `ui-user-questions` fijan el compositor rediseñado (multi-selección con casillas, fila personalizada siempre visible, paginador en el pie, re-traducción del feedback por clave de diccionario, Enter seguro para IME) y el registro de diccionarios del plugin más la cara de inyección; las pruebas de `ui-primitives` fijan el conjunto de iconos. La GUI Web ensamblada se ejercitó contra una sesión en vivo cubriendo las rutas de respuesta, cancelación e interrupción de turno.
