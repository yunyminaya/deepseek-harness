# Agent Note: Admisión atómica de imágenes Web

Status: implemented

[English](2026-07-29-atomic-web-image-admission.md) | [中文](2026-07-29-atomic-web-image-admission.zh.md) | Español

## Problema

La admisión de prompts con imagen y `session.selectModel` cruzan cada una búsquedas asíncronas de modelo y adjuntos. Sin un único punto de ordenamiento, un prompt con imagen podía validar un destino con capacidad de imagen mientras una selección concurrente instalaba un destino de solo texto. La selección también podía cambiar la ruta después de que la admisión hubiera comenzado pero antes de que se publicara el evento durable del mensaje.

## Decisión

Cada agente Web vivo tiene una cadena privada de promesas compartida por la admisión de prompts con imagen y la selección de modelo. Una operación fallida resuelve a su llamador normalmente y deja la cadena utilizable. Los prompts de solo texto se saltan la cadena porque no pueden crear este conflicto de ordenamiento.

La cadena da a las dos operaciones un orden determinista. Cuando la selección corre primero, la admisión de imagen posterior observa el modelo seleccionado y rechaza una imagen no soportada antes de la persistencia. Cuando la admisión de imagen corre primero, su adjunto y publicación de evento se completan antes de que la selección cambie la ruta. El runtime LLM compartido puede entonces proyectar bloques durables de imagen a marcadores de posición de texto deterministas para una solicitud de solo texto sin reescribir el registro de sesión. El steering usa la misma cadena de admisión aunque no entre en el espejo de UI encolado.

Los adaptadores de provider siguen siendo la frontera final de aplicación. El ordenamiento del host solo evita que su ruta mutable y su estado de imagen pendiente se contradigan entre sí antes del ensamblado de la solicitud.

## Alternativas consideradas

**Escanear el historial durable o derivado antes de la selección.** Esto impedía que se seleccionara una ruta de solo texto siempre que el historial contuviera una imagen. La proyección local a la solicitud ahora soporta esa ruta directamente, así que el historial ya no es una restricción de selección.

**Rastrear la publicación pendiente por separado.** Una ocurrencia encolada podía retenerse desde el dequeue hasta su evento coincidente. La cadena de promesas ya mantiene la selección detrás de la operación de admisión completa, así que un segundo espejo de ciclo de vida es innecesario.

**Serializar cada prompt y mutación de sesión.** Los prompts de solo texto y las operaciones de sesión no relacionadas no pueden introducir un requisito de imagen. Un lock más amplio añadiría latencia y propiedad sin cerrar otra carrera de modalidad.

## Consecuencias

Un prompt con imagen y una selección de modelo concurrente tienen orden determinista. La selección puede esperar la admisión de imagen en vuelo, mientras que los prompts de texto no relacionados conservan su concurrencia existente. La selección de modelo de solo texto sigue disponible después de que las imágenes entren al historial durable porque el ensamblado de la solicitud proyecta esas imágenes a marcadores de posición.
