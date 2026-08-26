# Agent Note: Plegar los hechos de sesión solo-para-traza en eventos que soportan carga

Status: implemented

[English](2026-06-20-collapse-trace-only-session-events.md) | Español

## Problema

El vocabulario de eventos de sesión incluye eventos de primera clase que no forman parte del historial de conversación reproducible y que tienen poca o ninguna consumición en producción. `usage` ya está presente como fragmento del stream del modelo antes de que el loop además añada un evento `usage` separado. `error` duplica el motivo de fallo del loop de `turn/end { kind: 'error', message, code }`; la liquidación de ACP lee el motivo de fin de turno, mientras que las proyecciones de mensajes y de UI se saltan el evento `error` independiente.

Estos eventos hacen que el transcript canónico parezca más útil como telemetría de lo que realmente es. Añaden variantes de evento, invariantes, pruebas, instantáneas y casos de persistencia, pero no soportan carga como registros separados. Los hechos que llevan aún pueden ser útiles: el uso de tokens debería seguir disponible para la contabilidad, y el número de paso de un error no debería desaparecer en silencio. La simplificación consiste en plegar esos hechos en eventos cercanos que los consumidores ya deben entender, no en registrar menos información.

## Decisión

Los eventos independientes solo-para-traza se eliminan exactamente donde su información se preserva sin un registro paralelo:

- El uso de un paso exitoso se pliega en el `assistant/message` correspondiente (`assistant/message { turn, step, content, usage? }`), de modo que la salida de modelo ensamblada y su contabilidad viajen juntas.
- Un paso fallido o abortado que tiene uso pero no contenido de asistente lleva el uso en un `assistant/message` de contenido vacío (`assistant/message { content: [], usage }`) — ningún fragmento de uso persistido queda sin representar. El caso sin pérdida de información es la vía de max-tokens: un paso cortado con uso pero contenido vacío (p. ej. solo una llamada de herramienta descartada) antes emitía un `usage` independiente. Para impedir que el evento de contenido vacío inyecte un turno de asistente sin contenido y espurio en el transcript del provider, `deriveMessages()` se salta los eventos `assistant/message` de contenido vacío; una prueba de regresión afirma que el uso sigue representado Y que el historial derivado permanece intacto.
- El número de paso del evento `error` independiente se pliega en `turn/end.reason` para `kind: 'error'` (`{ kind: 'error', step, message, code? }`) — `turn/end` es el resultado durable del turno que ACP y la reanudación ya consumen.
- `agent/error` y el registro de logs permanecen para el diagnóstico en vivo; no hay un segundo registro de error del log de sesión después de `turn/end`.

El log de conversación del usuario contiene lo necesario para renderizar, reanudar, auditar y contabilizar la interacción sin que los consumidores reconcilien filas de traza duplicadas.

## Alternativas consideradas

**Conservar las filas independientes como telemetría** — los eventos hacían que el transcript canónico pareciera más útil como telemetría de lo que era, al coste de variantes de evento, invariantes, pruebas, instantáneas y casos de persistencia que nada consumía. Si la analítica se vuelve real, la forma correcta es un helper de proyección o un almacén de telemetría dedicado con su propia política de retención — no filas de traza duplicadas en el log de conversación.

## Verificación

`SessionEventMap` no lleva ningún `usage` ni `error` independiente; el loop no añade un evento de uso separado y registra los fallos durables mediante `turn/end { kind: 'error', step, message, code? }`; las instantáneas de ACP y las pruebas de persistencia afirman que no hay líneas solo-para-traza; los fixtures grabados usan la nueva forma de evento con la versión de formato de sesión fijada en `0` (los backends rechazan cualquier log almacenado que no sea `0` según la política de formato pre-release); y la documentación indica dónde se observan el uso de tokens y los errores operativos.

## Consecuencias

Un consumidor ya no puede filtrar el log canónico por `usage` independiente o filas `error` a nivel de paso. Debe leer esos hechos de los eventos de asistente/fallo que los llevan. Es una simplificación razonable porque los mismos hechos siguen presentes, como demuestra la sección de Verificación.

## Nota de implementación

**Versión de formato.** Esto cambia los eventos persistidos, pero el formato de sesión pre-release permanece fijado en `0` y rechaza cualquier otra versión sin migración. `dsh-session` es propietario de la constante usada por los escritores y la validación de carga. Las versiones de formato monótonas comienzan en el primer release.

El uso ahora se observa en `assistant/message.usage`; el paso de un error operativo, en `turn/end.reason` para `kind: 'error'`. `agent/error` + el registro de logs no cambian para el diagnóstico en vivo.
