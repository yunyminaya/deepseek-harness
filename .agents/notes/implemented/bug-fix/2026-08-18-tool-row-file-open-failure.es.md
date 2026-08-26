# Agent Note: Los fallos de apertura de archivo de la fila de herramientas siguen visibles

Status: implemented

[English](2026-08-18-tool-row-file-open-failure.md) | Español

## Problema

Los clics de ruta de la fila de herramientas ya llaman a `host.openPath` a través del `openFile` inyectado por la vista de chat. El inject se tragaba cualquier rechazo del Host o del SO, de modo que un abridor de escritorio ausente, un carrier remoto o no-loopback, o una ruta que el Host no puede traspasar dejaban la fila con aspecto de éxito. El lector no tenía motivo ni segundo intento.

La [decisión de apertura de archivo en el SO](../feature/2026-07-28-tool-call-file-open-in-os.es.md) sigue siendo dueña del gesto de enlace y del traspaso al Host. Esta nota solo es dueña del rechazo.

## Decisión

El inject devuelve la promesa de `workspaces.openPath`. La vista de chat envuelve ese abridor: un rechazo abre un Modal en la página con el texto lanzado (o el respaldo de apertura desconocida cuando ese texto está vacío) y un botón Reintentar que repite la misma ruta; Cancelar, Escape, el control de cierre y un clic en la máscara lo descartan. Un cierre posterior al descarte se ignora, de modo que un rechazo en vuelo cancelado no puede reabrir el diálogo.

El diálogo vive en la vista que es dueña de la llamada al Host, no en cada fila de herramientas. Los chips de archivos producidos y las menciones del mensaje de cierre usan el mismo envoltorio porque ya comparten ese abridor. La acción de carpeta de archivos producidos abre `.`, y ese rechazo usa el título de carpeta y el texto de apertura desconocida.

El mensaje del Host se muestra tal como se lanza. `WorkspaceRuntime.openPath` antepone `path open failed: ` al error del wire; el diálogo no desenvuelve ese prefijo.

## Alternativas consideradas

- **Error inline por fila.** La llamada al Host pertenece a la conversación y varias entradas comparten un mismo abridor; un banner local a la fila duplicaría el mismo rechazo junto a cada objetivo de clic.
- **Toast sin reintento.** La petición del producto es la razón *y* una entrada de reintento. El diálogo de adopción de carpeta del workspace ya empareja esas dos.
- **Persistencia de remontaje en el chat-store.** Una apertura fallida es estado de vista transitorio. El chat-store sobrevive a los remontajes de vista, así que un diálogo sobrante volvería tras un cambio de pestaña que no puede reintentar útilmente el gesto original.

## Consecuencias

Un rechazo silencioso del Host ya no parece un éxito desde el asiento del lector. Los despliegues headless o remotos que hacen clic en una ruta ven ahora por qué no se produjo el traspaso al escritorio. La vista guarda un contador extra de generación de peticiones para que el descarte y el reintento sigan siendo seguros frente a carreras.

## Pruebas

Las specs del paquete cubren el rechazo del inject, el texto del diálogo (Error, no-Error, vacío, carpeta de workspace), el reintento de la misma ruta, la cancelación y un cierre que llega después del descarte. `apps/web/tests/seeded-history.e2e.ts` hace que `host.openPath` falle sobre una fila de lectura reanudada en frío, fija el diálogo ensamblado en `file-open-failure.expected.md` y verifica la razón en inglés más una segunda llamada con el mismo payload.
