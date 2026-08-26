# Agent Note: Cifras de la franja de estadísticas de sesión completa mediante una proyección sessionStats

Status: implemented

[English](2026-08-12-full-session-turn-step-counts.md) | [中文](2026-08-12-full-session-turn-step-counts.zh.md) | Español

## Problema

La franja de estadísticas del chat web plegaba la ventana de conversación cargada de `StatsLine` (`deriveStats` sobre `chat.legacy.nodes`) para cada cifra sin tokens: el contador «N turnos · M pasos», los tiempos de pared de LLM y herramientas, y las medias de TTFT/régimen. El historial se pagina de 50 mensajes en 50, así que cada clic en 加载更早 (Cargar anteriores) agrandaba la ventana y con ella cada cifra — 7 turnos · 44 pasos pasaba a 10 turnos · 89 pasos tras una página, y la duración de LLM subía igual. La expectativa del producto son cifras de la sesión completa, independientes de cuánto historial haya cargado un cliente. La contabilidad de tokens en la misma franja ya tenía la arquitectura correcta: la proyección durable `tokenUsage`.

## Decisión

Un nuevo plugin funcional `@deepseek-ai/dsh-session-stats` registra una unidad de proyección `sessionStats` en `ctx.sessionProjections`, montada como fila de bundle de la web-app. El valor porta el conjunto completo de cifras sin tokens de la franja — `{ turns, steps, llmMs, toolMs, ttftMs, ttftSteps, decodeMs, decodeTokens }`, con nombres de campo que reflejan el pliegue de ventana para que ambos se intercambien íntegros. `steps` cuenta eventos `step/end` y `turns` cuenta turnos distintos que llevan al menos uno (los números de turno son monótonos, así que basta un slot `lastTurn`); `llmMs` suma `step/start` → `assistant/message`; el TTFT registra el primer fragmento de delta no vacío por paso (sobreviviendo al `llm/retry` dentro del paso, la paridad del `resetForRetry` de la ventana); la decodificación abarca del primer token al mensaje ensamblado en los pasos que reportan uso; `toolMs` empareja `tool/call` → `tool/result` por callId, descartando las llamadas sin resolver en `turn/end`. El predicado de primer token `isTokenDelta` se movió a `@deepseek-ai/dsh-llm/message` (junto al tipo `StreamChunk` que discrimina) para que el pliegue del host y el índice de tiempos del cliente compartan una implementación; client-runtime lo reexporta. La entrega es enteramente el seam de proyecciones existente — bloque de página de cola de historial, tramas push de `session/projection`, filas de listado — sin cambios en apiproxy, esquemas del cable ni el runtime de cliente. `StatsLine` lee `useProjection('sessionStats')` y recurre al pliegue de ventana cuando la clave está indefinida (un ensamblado sin la unidad). El fixture de conexión de cliente refleja el pliegue como `sessionStatsOf` bajo su disciplina existente de clave-por-ensamblado-compuesto.

`step/end` — no `assistant/message` — es el evento contado, por dos razones de corrección halladas al revisar el diseño obvio de conteo de mensajes:

1. Un paso con max-tokens añade un `assistant/message` de contenido vacío que existe solo para alojar el uso y nunca llega a la superficie; contar mensajes contaría un paso que el transcript no muestra.
2. Un paso cancelado aborta antes de que su mensaje se ensamble (ningún `assistant/message` en absoluto), y sin embargo el cliente sintetiza un nodo asistente interrumpido visible; contar mensajes descartaría silenciosamente pasos cancelados comunes.

`step/end` se añade exactamente una vez por paso iniciado, en el `finally` del bucle, así que los pasos completados, fallidos, cancelados y con max-tokens dejan todos uno — y el contador avanza en el asentamiento del paso, el mismo momento en que avanzaba el pliegue de ventana, de modo que el comportamiento en vivo no se desplaza.

## Alternativas consideradas

**Contar eventos `assistant/message`.** Rechazado por los dos defectos de corrección anteriores (sobreconta mensajes que alojan uso, subconta pasos cancelados).

**Contar eventos `step/start`.** Cobertura equivalente (precede a cada `step/end`), pero el contador avanzaría al comenzar un paso en lugar de al asentarse — un cambio visible de comportamiento en vivo sin beneficio; la ubicación en el `finally` del `step/end` da la misma completitud.

**Registrar la unidad en `core/agent-loop` (el productor de eventos).** El bucle es la espina dorsal del producto; un modelo de lectura de UI allí añade una dependencia de session-projection a cada ensamblado, contra «plugins, no cambios al bucle» y «mantener los opt-ins fuera de los valores por defecto publicados».

**Registrar la unidad en `token-meter` (un pliegue existente sobre los mismos eventos).** Contar turnos/pasos no es medir tokens; cada clave de proyección vive en el paquete dueño de su dominio.

**Plegar el registro completo en el cliente.** El cliente solo retiene por diseño la ventana paginada; la regla de sin-pliegue-en-cliente del RFC de proyecciones existe justamente para que las cifras sobrevivan a la paginación, la compactación y las lecturas en frío.

**Mantener tiempos de pared, TTFT y régimen con ámbito de ventana, leyéndolos como «lo que hay en pantalla».** Rechazado: la misma queja de paginación aplica a la duración de LLM, y una franja que mezcla conteos de registro completo con tiempos de ámbito de ventana se lee como un conjunto de cifras inconsistente. La proyección porta el conjunto completo, con el pliegue de ventana degradado a fallback sin unidad.

## Consecuencias

La franja muestra cifras del registro completo desde la primera página de cola; la paginación deja fijos todos los grupos. Las diferencias definidas de los bordes respecto a la semántica antigua de ventana están documentadas en el README del paquete: un paso que no produjo salida visible (falló antes del contenido) sigue contando, un paso interrumpido por un fallo cuenta una vez que la recuperación lo cierra con un `step/end` sintético al recargar (`interruptedTurnClosers`), un paso cancelado cuenta pero no aporta tiempo de pared (ningún mensaje ensamblado), y un mensaje con max-tokens que aloja uso aporta tiempo de modelo que la superficie no muestra. Cada página de cola web y cada fila de listado lleva una clave pequeña más, y el estado interno de la unidad cambia en fronteras de paso y fragmentos de primer token, así que el feed de cambios emite unas pocas tramas de valor idéntico por paso; los ensamblados TUI y headless no sirven ninguna clave `sessionStats` y cualquier consumidor recurre al pliegue de ventana. Dos sondas e2e que habían interpretado la franja como medida de ventana cargada (`chat-scroll-contract`, `complex-history.perf`) ahora cuentan filas de flujo montadas / pies de cola de turno. El escenario web `stats-paged-history` siembra en frío un registro de 28 turnos y fija que la franja completa lea totales completos en una página de cola parcial y que no se mueva al pulsar Cargar anteriores.
