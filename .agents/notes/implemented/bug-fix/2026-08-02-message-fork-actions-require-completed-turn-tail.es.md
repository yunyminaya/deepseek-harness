# Agent Note: Las acciones de bifurcación de mensaje requieren una cola de turno completado

[English](2026-08-02-message-fork-actions-require-completed-turn-tail.md) | [中文](2026-08-02-message-fork-actions-require-completed-turn-tail.zh.md) | Español

Status: implemented

## Problema

La conversación web adjuntaba la bifurcación al último nodo de assistant con texto no vacío de cada turno. Un resultado de herramienta posterior, un nodo de razonamiento interrumpido o un error terminal no asumían la propiedad porque esas filas no tienen IconActions de texto de contenido. El icono de bifurcación podía aparecer por tanto bajo una respuesta de assistant mientras quedaban más filas del mismo turno debajo. El Host expandía correctamente ese ancla de mensaje hasta el `turn/end` contenedor, pero la colocación hacía que la acción pareciera un corte a nivel de mensaje y el hijo heredaba visiblemente el sufijo del mismo turno.

## Decisión

`ConversationSnapshot.turnEnds` conserva los límites de turno completado presentes en la ventana de eventos en bruto. La vista de conversación recorre los nodos del transcript a través de cada límite y habilita la bifurcación solo cuando el último nodo del límite es un mensaje de usuario, un mensaje de steering durable o un mensaje de assistant con contenido. Los turnos abiertos no tienen ningún mensaje elegible, y un resultado de herramienta posterior, una interrupción solo de razonamiento, un error de turno u otro nodo del transcript dejan la bifurcación no disponible en mensajes anteriores. El control no disponible permanece visible, enfocable y con hover; `aria-disabled`, un tooltip y `aria-describedby` explican el requisito de cola completada sin enviar una petición al Host. Copiar y el reloj siguen disponibles bajo su chrome de mensaje existente, y la semántica de bifurcación de turno completado del Host permanece sin cambios.

La mitad de burbuja-de-mensaje de esta elegibilidad queda sustituida por la [eliminación de la bifurcación en las burbujas de usuario](../simplification/2026-08-06-user-bubbles-drop-the-branch-action.es.md): las burbujas de usuario y de steering ya no renderizan el control en absoluto, por lo que solo las colas de assistant con contenido pueden bifurcarse; la guarda del lado del assistant y su presentación visible-pero-no-disponible se mantienen.

Esto restringe la elegibilidad de mensaje establecida por la [decisión anterior de acciones de bifurcación de sesión web](../feature/2026-07-27-web-session-fork-actions.es.md). La bifurcación a nivel de fila de sesión sigue seleccionando el último turno completado, y las acciones de mensaje elegibles siguen pasando su seq de evento a través de la operación compartida del runtime del cliente.

## Alternativas consideradas

**Cortar el registro de eventos en el mensaje de assistant clicado.** Rechazado porque un mensaje de assistant puede estar dentro de un paso abierto y contener llamadas de herramienta cuyos resultados ocurren después. Un prefijo en bruto en ese seq no es un turno equilibrado y puede no ser un transcript de provider válido.

**Inferir la finalización a partir de `running` o del siguiente mensaje de usuario.** Rechazado porque los turnos de reintento y de steering no tienen por qué alinearse con la siguiente burbuja de usuario visible, y una ventana paginada puede omitir esa burbuja posterior. El evento durable `turn/end` es el hecho de finalización autoritativo.

**Ocultar la bifurcación de todo turno interrumpido.** Rechazado porque un turno abortado está durablemente cerrado y su texto interrumpido final puede ser la cola real del transcript. La elegibilidad depende del límite completado y del orden de nodos, no del tipo de resultado.

**Ocultar los controles de mensaje no elegibles.** Rechazado porque un control que desaparece no explica el requisito de límite y desplaza un chrome de mensaje por lo demás estable. Un control no disponible pero enfocable conserva la affordance mientras impide la petición.

## Consecuencias

Un icono de bifurcación habilitado denota el mismo límite de turno completado que el Host copiará. En la forma informada de respuesta → herramienta → Think interrumpido, la respuesta conserva copiar, el reloj y un control de bifurcación deshabilitado que explica por qué no puede actuar. Este cambio no proporciona deliberadamente edición de transcript dentro del mismo turno ni una operación de reintento-antes-del-turno; la acción de fila de sesión sigue disponible cuando un lector quiere copiar íntegro el último turno completado. Las pruebas de runtime fijan la proyección de límites y la estabilidad de referencias, mientras que las pruebas de conversación cubren colas de assistant, solo de usuario y de steering durable, además de los controles no disponibles causados por filas posteriores de herramienta y de razonamiento interrumpido.
