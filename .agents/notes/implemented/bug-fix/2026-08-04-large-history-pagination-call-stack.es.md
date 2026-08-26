# Agent Note: La procedencia de historial grande se escanea sin expansión de argumentos

Status: implemented

[English](2026-08-04-large-history-pagination-call-stack.md) | Español

## Problema

Un mensaje de asistente finalizado puede referenciar cientos de miles de chunks transmitidos a través de `sourceEventSeqs`. La paginación del historial buscaba el primer evento del grupo del mensaje con `Math.min(event.seq, ...sourceEventSeqs)`, de modo que una sesión válida podía exceder el límite de argumentos de función del motor de JavaScript y hacer que `session.history` fallara con HTTP 500.

## Decisión

La paginación escanea `sourceEventSeqs` y actualiza el número de secuencia más temprano un elemento a la vez. El algoritmo sigue siendo lineal en el tamaño de la procedencia y preserva el límite de página existente: una página comienza antes que todas las fuentes registradas de su mensaje más antiguo incluido.

Una prueba de regresión rechaza las llamadas de mínimo con múltiples argumentos y verifica que cada evento de procedencia permanece en la página junto a su mensaje finalizado. Esto ejercita el mecanismo del fallo sin que la suite de pruebas por defecto tenga que asignar un flujo de chunks de tamaño de producción.

## Alternativas consideradas

- **Subir el límite de pila o de argumentos de JavaScript** — rechazada: el límite depende del motor y del despliegue, y la expansión del arreglo sigue haciendo que un historial válido dependa de un techo de runtime ajeno.
- **Truncar `sourceEventSeqs` durante la paginación** — rechazada: podría cortar una página dentro de un mensaje y violar la agrupación de la reproducción.
- **Acotar el número de chunks transmitidos en la frontera del provider** — rechazada: los providers pueden emitir legítimamente flujos largos, y la paginación debe manejar toda representación válida de sesión.

## Consecuencias

- Los arreglos de procedencia grandes ya no hacen que la paginación del historial lance una excepción solo por su longitud.
- Las semánticas de paginación y las respuestas del protocolo no cambian.
- Esto no acota el tamaño en bytes de una página de historial ni el coste de reproducirla en el navegador; esas preocupaciones de rendimiento siguen siendo ajenas al fallo de pila de llamadas del lado servidor.
