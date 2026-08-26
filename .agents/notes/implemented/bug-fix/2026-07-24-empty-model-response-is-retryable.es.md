# Agent Note: Las respuestas vacías del modelo son fallos EMPTY_RESPONSE reintentables

Status: implemented

[English](2026-07-24-empty-model-response-is-retryable.md) | Español

## Problema

Los provider en ocasiones devuelven una respuesta degenerada: un flujo bien formado que lleva un final `stop` terminal y cero bloques de contenido — sin texto, sin razonamiento, sin llamadas de herramienta. Si un adaptador asigna esta forma a un final `{kind: 'stop'}` exitoso, el agent loop registra un `assistant/message` vacío y termina el turno como `completed`. El reintento nunca se ejecuta, ningún fallo llega a quien llama, y un driver como goal-round-driver consume una ronda sin progreso.

## Decisión

Un adaptador clasifica una respuesta completada vacía como un fallo del límite del provider, y la política de reintento la trata como transitoria:

- `dsh-llm` exporta el código canónico `EMPTY_RESPONSE_CODE` (`'EMPTY_RESPONSE'`) junto a `CONTEXT_WINDOW_EXCEEDED_CODE`/`QUOTA_EXCEEDED_CODE`.
- `dsh-llm-pi-ai` (`mapStopReason`): un `stop` terminal cuyo mensaje de asistente no tiene bloques de contenido se convierte en un `finish {kind: 'error'}` con ese código. La detección de desbordamiento de contexto sigue ganando donde aplica (se comprueba primero y es la clasificación más accionable).
- `dsh-llm-deepseek` (`translate`): en `[DONE]`, un final `stop` (o ausente) sin bloques abiertos se convierte en el mismo final de error. Los flujos de solo razonamiento cuentan como contenido y siguen siendo exitosos.
- El valor por defecto de reintento normal propiedad del provider incluye `EMPTY_RESPONSE`: el intento no produjo nada duradero, así que repetirlo es seguro; los despliegues pueden eliminarlo vía `retryableCodes`, y `dsh-llm-retry` ejecuta la política resuelta.

La detección se limita a los finales `stop` solamente. `max-tokens` con contenido vacío conserva su significado existente (pi-ai ya normaliza el caso de desbordamiento sin salida), `tool-calls` no puede carecer de bloques en la práctica, y los finales de error/abortados ya fallan.

La clasificación usa la maquinaria existente del agent loop — `finishError` → `agent/request-error` → `dsh-llm-retry` — y mantiene `agent-loop` neutral respecto al provider. Agotar el presupuesto de reintentos termina el turno con un fallo `EMPTY_RESPONSE` explícito en lugar de un éxito vacío.

## Alternativas consideradas

**Detectarlo en el agent loop o en `BlockAssembler`.** Una implementación compartida, pero mueve el juicio sobre la respuesta del provider hacia el agent loop, contra «plugins, no cambios en el agent loop», y el ensamblador es un algoritmo puro de ensamblado. El adaptador es donde los hechos del protocolo se convierten en clasificación del harness, con la reclasificación de desbordamiento como precedente exacto.

**Un plugin de transformación de flujo en el waterfall de `llm/stream`.** Neutral respecto al provider y de implementación única, pero añade un paquete más el cableado para lo que es un hecho del límite que cada adaptador puede declarar en pocas líneas, y el comportamiento activado por defecto seguiría exigiendo tocar cada bundle.

**Tratar también como vacías las respuestas de solo espacios o de solo razonamiento.** Rechazada por excesiva: esas respuestas llevan contenido producido por el modelo, y clasificar mal una respuesta legítima (aunque inútil) como fallo de clase transporte arriesga bucles de reintento en modelos que se detienen intencionadamente tras razonar. El alcance es exactamente «cero bloques de contenido».

## Consecuencias

- Un provider con un mal funcionamiento transitorio consume un reintento acotado en lugar de un turno sin salida; un modelo persistentemente vacío produce un fallo de turno `EMPTY_RESPONSE` accionable.
- Un modelo que genuinamente pretende no decir nada (raro, pero posible tras un resultado de herramienta) se reintenta y, si persiste vacío, falla el turno. Este intercambio se aceptó deliberadamente: un mensaje de asistente vacío es indistinguible del defecto del provider y no tiene valor para el usuario.
- La instantánea ACP `empty-response-retry` (un escenario sin clave redactado con un overlay de reintento determinista de 1 ms sin jitter, `examples/acp-agent/retry.cordis.yml`) fija el comportamiento visible del producto: un evento `llm/retry` duradero, sin salida ACP para el intento descartado, la respuesta recuperada y un turno completado limpio.
