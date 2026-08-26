# Agent Note: Structured error taxonomy

Status: implemented

[English](2026-06-11-structured-error-taxonomy.md) | Español

## Problem

Los fallos cruzaban seams como cadenas desnudas. Un error de herramienta se aplanaba a un bloque de texto — nombre, código y pila perdidos — por lo que un complemento futuro de sandbox/reintentos no podía distinguir ENOENT de EACCES, y el modelo recibía menos retroalimentación accionable de la posible. Un lanzamiento que no era un Error se degradaba aún más: el bucle lo envolvía en `new Error(String(x))`, eliminando cualquier código. Y `LlmError` era el único error tipado en el sistema, sin una base compartida, por lo que no había nada con lo que un Consumer pudiera hacer `instanceof` de forma genérica.

## Decision

Una sola base `HarnessError extends Error` en `dsh-llm` (el paquete hoja que todos los demás importan — sin nueva arista de dependencia): un `code` estable distinto de `message`, encadenamiento `cause` a través de `ErrorOptions`, y `name` que predetermina a la subclase. `isHarnessError` restringe en seams.

- `LlmError` y `ToolArgsError` (dsh-tools) la extienden, manteniendo sus códigos existentes.
- `ToolExecutionResult` gana `error: { name, code }` opcional, poblado en el catch del registro cuando el valor lanzado es un `HarnessError`. El bucle del agente lo reenvía al evento de sesión `tool/result` (que ganó el mismo campo opcional), por lo que el fallo estructurado sobrevive en el registro para complementos de reintento/sandbox y repetición. El bloque de texto orientado al modelo permanece sin cambios.
- El `toError` del bucle envuelve un lanzamiento que no es Error en un `HarnessError` (`code: 'UNKNOWN'`, original encadenado como `cause`) en lugar de un `Error` desnudo, por lo que incluso un lanzamiento incorrecto lleva un código enrutable al evento de sesión `error` (que ya exponía `code`).

## Consequences

- Los errores son enrutable-máquina de extremo a extremo: un complemento puede ramificar según `error.code` en lugar de hacer coincidir por subcadena un mensaje.
- Una sola clase base se importa ampliamente, pero vive en el paquete del que todos ya dependen, por lo que el costo es una sola importación, no una nueva arista.
- `deriveMessages` no expone `error` en el historial del modelo — el modelo todavía ve el bloque de texto; el campo estructurado es para código y repetición.
- La validación de argumentos retiene su código y comportamiento existente; las invariantes de diagnóstico propias del paquete llevan su código estable de forma independiente, por lo que el registro de invariantes no importa un paquete de producto. La base compartida añade metadatos de enrutamiento entre seams sin cambiar el texto orientado al modelo.

<!-- agent-note-format: alternatives-not-recorded (pre-format Agent Note) -->