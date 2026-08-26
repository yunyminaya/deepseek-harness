# Agent Note: Eliminar el stub de edición del mensaje de usuario

Status: implemented

[English](2026-07-31-drop-user-message-edit-stub.md) | Español

## Problema

La fila IconActions de la burbuja de usuario llevaba un botón de edición junto a copiar y ramificar. Nada lo respaldaba: el control no tenía manejador de clic, ni mutación de cliente, ni operación del host para reenviar un mensaje editado. Quien lo encontraba veía un control que prometía una capacidad que el producto no puede honrar.

## Decisión

`MessageIconActions` renderiza solo reloj / copiar / ramificar, y su prop `edit` desaparece junto con el botón; `MessageItem` ya no la pasa. La burbuja de usuario y el marco del asistente ahora difieren solo en el lado del reloj. El README del paquete registra la capacidad ausente en Known Limitations, y la prueba golden de acciones de mensaje de la web fija la fila sin el control.

El locale común conserva su término genérico `edit`, que es vocabulario compartido y no el texto propio de este componente.

Reintroduce el control junto con la capacidad: una mutación de cliente que edite un mensaje de usuario ya asentado y el comportamiento del host que decida qué hace el mensaje editado con el turno que ya lo consumió.

## Alternativas consideradas

**Desactivar el botón con un tooltip.** Un control visible pero muerto sigue anunciando la edición y cuesta lo mismo explicar; la eliminación es el estado honesto.

**Conectarlo al editor de la cola.** La cola edita un mensaje que aún no se ha enviado. Un mensaje de usuario asentado ya está en el transcript y en el contexto del modelo, así que reutilizar ese editor significaría, en silencio, otra cosa.

## Consecuencias

La web no ofrece forma de corregir un mensaje enviado; ramificar desde el mensaje es el gesto más cercano disponible. La reintroducción es un cambio solo de UI cuando exista la mutación, porque la fila compone sus acciones a partir de props.
