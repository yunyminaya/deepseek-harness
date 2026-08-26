# Agent Note: Ciclo de vida del despacho en vivo de Code Mode y paralelismo con contrato nativo

Status: implemented

[English](2026-07-26-code-mode-live-parallel-dispatch.md) | Español

> Ámbito: el evento `tool/code-dispatch-start`, el estado de ejecución por sub-call en el chat web, y el scheduler del bridge reutilizando el contrato de concurrencia nativo. Se construye sobre la [base del host](2026-07-26-code-dispatch-ui-foundation.es.md) y las [filas de sub-call de chat](2026-07-26-code-mode-chat-subcall-rows.es.md); el contrato nativo en sí es propiedad de la [nota de llamadas de herramienta en paralelo](2026-07-10-parallel-tool-call-execution.es.md).

## Problema

Quedaban dos vacíos después de que se enviaran la base del host y las filas de sub-call de chat. Las filas de sub-call solo aparecían cuando cada despacho se *asentaba* — mientras uno se ejecutaba, la UI no mostraba nada para él, así que un sub-call lento se leía como un padre bloqueado. Y el bridge serializaba cada llamada de binding («incluso `Promise.all` ejecuta de una en una»), un marcador de posición de antes de que las herramientas llevaran metadatos de concurrencia: `isConcurrencySafe` ya existe, el scheduler del loop ya ejecuta hermanos nativos en pools acotados, y un programa de Code Mode que espera tres lecturas independientes pagaba 3× la latencia que pagaría la ruta nativa.

## Decisión

**Un par de ciclo de vida, un contrato de scheduling, compartido con el nativo.**

- **Par de eventos**: `tool/code-dispatch-start` (ids de padre/sub, nombre, args normalizados) se añade cuando el scheduler realmente inicia una llamada — no en el envío, de modo que una llamada en cola abandonada por el asentamiento de la ejecución no registra nada. El `tool/code-dispatch` existente asienta el par (mismo `subCallId`); cada llamada iniciada se asienta exactamente una vez (los aborts asientan como resultados `isError` a través del pipeline). El timing = los campos `time` de los dos eventos. Ambos siguen siendo solo de log; el contexto de modelo no se toca; el formato sigue en v0.
- **Scheduler del bridge**: las llamadas enviadas se clasifican en el momento del inicio vía `registry.executionMode` (el MISMO contrato `isConcurrencySafe` fail-closed que usa el loop) y se inician estrictamente en orden de envío. Un único driver de un carril es dueño de cada etapa ORDENADA — el append de inicio, `prepare` (pre-execute/guards), el commit head-of-line `finalize`/`finish` (post-execute + diferimiento de contexto + append de asentamiento) — de modo que las etapas de política ordenadas nunca se solapan entre sí y solo la etapa around-dispatch/body se ejecuta concurrentemente, exactamente la secuenciación del loop nativo (`fillPool` espera `startCall` y luego `commitReady`). Las llamadas consecutivas clasificadas como paralelas se solapan hasta `maxParallelSubCalls` (un campo de `Config` validado por el schema del Loader Y revalidado en la construcción directa, por defecto 10 — el propio default del scheduler del loop; `1` restaura el despacho serial); una llamada exclusiva drena el pool, se ejecuta sola y mantiene su barrera hasta que su COMMIT se completa (post-execute incluido), como un grupo exclusivo nativo. El asentamiento de la ejecución aborta los despachos en vuelo y abandona los encolados no iniciados (rechazo de binding, sin eventos), y luego drena hasta la quietud — incluido un commit ya en pleno vuelo cuando el programa devolvió — antes de que el resultado externo cierre el turno.
- **Cliente**: el `ToolCallTree` del Runtime almacena un evento de inicio como hijo `RunningToolCall` y lo proyecta a través de los `subCalls` recursivos del padre (las filas derivan el anillo de ejecución de esa forma, exactamente como para las llamadas nativas en vuelo). Su asentamiento reemplaza la entrada del índice privado en su lugar, preservando el orden de inicio bajo la finalización en paralelo y llevando el `time` del inicio como `callTime` (fuente de duración). Un asentamiento sin inicio observado (ventana cortada a mitad del par, o un log anterior al evento de inicio) se añade directamente, de modo que los logs antiguos siguen renderizando.
- **Prompt del SDK**: la frase orientada al modelo «las llamadas se ejecutan secuencialmente» se reemplaza por el contrato verdadero (las llamadas seguras independientes pueden solaparse bajo `Promise.all`; el trabajo dependiente se secuencia con `await`) — un cambio visible para el modelo, re-grabado en cada instantánea de code-mode.

## Alternativas consideradas

**Paralelismo sin restricciones (dejar que `Promise.all` lo solape todo).** Rechazada: las escrituras podrían competir; el scheduler nativo existe precisamente porque la herramienta, no el llamador, es dueña de la afirmación de seguridad. Un único vocabulario de concurrencia entre nativo y Code Mode era el requisito asentado.

**Emitir el evento de inicio en el envío en lugar de en la entrada al pool.** Rechazada: un inicio en el momento del envío mostraría las llamadas encoladas pero nunca ejecutadas como «en ejecución» y forzaría un tercer evento terminal «abandonado» para reconciliar el log. El inicio en la entrada mantiene el invariante *iniciado ⇔ se asienta exactamente una vez* y no necesita ningún tercer evento.

**Reutilizar directamente la implementación del scheduler del loop.** Rechazada: el loop programa un lote completamente parseado con commit de resultados en orden de modelo; el bridge programa un flujo abierto de envíos cuyos resultados vuelven al programa (no al transcript), de modo que solo se comparte el *contrato* (clasificación, pool, barreras), no la maquinaria.

## Consecuencias

Los programas obtienen latencia de grado nativo para lecturas independientes sin API nueva en el lado del modelo — `Promise.all` simplemente funciona mejor, y la guía de prompt cambió en consecuencia. La UI web muestra anillos de ejecución por sub-call en vivo (el fixture emite pares start/settle; jsdom fija la forma en ejecución; la spec de runtime fija el asentamiento en el lugar, la finalización fuera de orden y el emparejamiento de callTime). Los tramos de sub-call de Trajectory/waterfall dibujan un timing veraz a partir del par. El acotado de spill ([spill del log de despacho](2026-07-26-code-dispatch-log-spill.es.md)) hereda el evento de asentamiento como su único punto de acotado.
