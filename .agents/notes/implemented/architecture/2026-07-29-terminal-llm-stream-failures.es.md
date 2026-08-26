# Agent Note: Fallos terminales del stream de LLM

Status: implemented

[English](2026-07-29-terminal-llm-stream-failures.md) | [中文](2026-07-29-terminal-llm-stream-failures.zh.md) | Español

Esta nota supera únicamente la identidad del error lanzado y el mecanismo de sidecar local a la llamada de la [recuperación acotada de peticiones al LLM](2026-06-21-bounded-llm-request-recovery.es.md) y de la [recuperación de desbordamiento de contexto tras la llamada](2026-07-10-after-call-compaction-pressure-and-overflow-recovery.es.md). Esas notas siguen siendo dueñas de los hechos de fallo estructurados, la política de reintentos, los intentos durables y la recuperación por compactación.

## Problema

Un fallo del adaptador tenía dos representaciones públicas: una excepción procedente de la selección, el despacho, la construcción del iterador o la iteración, y un `finish { kind: 'error' | 'aborted' }` en banda. `LlmRuntime` etiquetaba los objetos lanzados en un sidecar indexado por stream para que el agent loop pudiera distinguirlos de los fallos del middleware y de los consumidores. El consumidor seguía necesitando un catch alrededor de la iteración, las comprobaciones de señal, el registro de chunks y el ensamblado; la corrección dependía por tanto de demostrar qué sentencia lanzaba y de consultar los metadatos adjuntos al iterable devuelto exacto.

La política de reintentos tenía la misma propiedad indirecta. Se descubría a través del sidecar del stream después del despacho, aunque `prepareCall()` ya había capturado el registro servidor. Una ruta propiedad del wrapper y una ruta propiedad del adaptador compartían por consiguiente una única API de búsqueda opaca pese a tener autoridades distintas.

## Decisión

`LlmRuntime` es la frontera de normalización de un intento de adaptador. Captura solo la selección del adaptador final, el despacho síncrono, la construcción del iterador y los fallos de `next()`, convierte el valor lanzado en un `LlmFailure` inmutable y emite un único `finish` terminal. La cancelación del llamante o un fallo `ABORTED` seleccionan la razón `aborted`; cualquier otro fallo del adaptador selecciona `error`. Un adaptador también puede emitir cualquiera de las dos razones terminales directamente.

El catch propiedad del adaptador termina antes de cada chunk generado. Los errores del middleware de `llm/stream`, las llamadas anidadas, la limpieza del adaptador, los consumidores de chunks, el registro, las comprobaciones de señal y el ensamblado siguen lanzándose como defectos o fallos de ciclo de vida; nunca entran en la recuperación de la petición de modelo. Un fallo de transporte después de deltas parciales puede dejar bloques abiertos, por lo que el invariante del stream permite bloques abiertos solo para finalizaciones de error terminal o `aborted`. De esa salida incompleta no se ensambla ningún mensaje de asistente ni llamada de herramienta.

`PreparedLlmCall` expone la política de reintentos inmutable capturada con su config y su registro. La reutilización de un solo uso y el desajuste de config siguen siendo errores síncronos de uso indebido `INVALID_PREPARED_CALL`. Una ruta servida por completo por el middleware de `llm/stream` no tiene registro preparado y por tanto ninguna política servidora.

El agent loop consume una única representación de fallo. Itera y registra chunks sin un catch de clasificación, inspecciona el `finish` terminal y pasa sus hechos de fallo más la política preparada a `agent/request-error`. Las APIs sidecar públicas `isLlmAdapterFailure`, `llmFailureOf` y `llmRetryPolicyOf` están ausentes.

## Alternativas consideradas

**Conservar el etiquetado de errores local a la llamada.** Esto preserva la identidad del objeto lanzado, pero hace que cada consumidor capture una región que contiene su propio trabajo propenso a fallos y acopla la clasificación a la identidad de un wrapper de iterable. El objeto de error original no tiene ningún papel durable en la recuperación; los hechos normalizados son el valor de frontera útil.

**Exigir que cada adaptador emita chunks de fallo y prohibir los lanzamientos.** Los iteradores de librerías, los transportes y el despacho de JavaScript pueden seguir lanzando. Exigir que cada adaptador reproduzca la misma frontera de catch duplica la propiedad y no protege a un consumidor directo de `LlmRuntime` de una implementación incompleta.

**Capturar cada error de iteración en el agent loop.** El loop no puede distinguir de forma fiable un fallo del provider de uno del middleware, de la escritura de sesión, de la cancelación o del ensamblado sin restaurar un mapa sidecar de los objetos de stream a las llamadas de adaptador que los crearon. La clasificación pertenece a donde se hace la llamada al adaptador.

**Devolver un `Result` antes de transmitir.** Un resultado previo al stream no puede representar un fallo de transporte después de la salida parcial sin añadir un segundo ciclo de vida de respuesta. El chunk terminal existente ya representa tanto los resultados tempranos como los tardíos del intento.

## Consecuencias

Todos los consumidores de `LlmRuntime.stream()` reciben los fallos operativos del adaptador a través de un único protocolo terminal tipado, mientras que los fallos de programación y de ciclo de vida conservan la semántica ordinaria de excepción. La recuperación renuncia a la identidad exacta del objeto lanzado y expone solo hechos desvinculados y neutrales respecto al provider. El servicio de stream posee algo más de cableado de adaptador, pero los consumidores eliminan los catches que identifican qué adaptador lanzó y eliminan los metadatos indexados por stream. Las llamadas preparadas llevan su política explícitamente, y el enrutamiento solo de middleware sigue visiblemente sin política.
