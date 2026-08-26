# Agent Note: Renderizar las cadenas de causa de los errores en cada frontera de diagnóstico

Status: implemented

[English](2026-07-20-error-cause-chain-diagnostics.md) | Español

## Problema

Una ejecución de TUI contra un endpoint de DeepSeek inalcanzable falló con el único aviso `fetch failed` y sin más detalle. Dos carencias independientes produjeron ese callejón sin salida:

1. El `fetch` de undici envuelve todo fallo de transporte (DNS, conexión rechazada, TLS, proxy) en un `TypeError: fetch failed` desnudo cuyo detalle accionable — `ECONNREFUSED`, `bad port`, el AggregateError de Happy Eyeballs — vive en `error.cause`. Cada frontera de diagnóstico del harness renderizaba solo `error.message` (o `String(error)`, que es equivalente para los Errors), así que el envoltorio enmascaraba el diagnóstico en el aviso de la TUI, en la razón perdurable de `turn/end` y en cada línea de logger.

2. El punto de entrada readline (`dsh-stdio`) no renderizaba ninguna razón de fallo: un `turn/end` con `reason.kind === 'error'` no imprimía nada salvo el siguiente prompt `> `, así que el mismo fallo en `demo:repl` era silencio puro.

## Decisión

- `dsh-llm` exporta `errorChain(value)`: renderiza un valor lanzado con su cadena completa de `cause` (`outer: inner: …`) y los miembros de AggregateError (`msg [m1; m2]`), con contención de causas circulares y coerción hostil. Es solo un renderizador de salida de diagnóstico; el enrutado sigue en `HarnessError.code`.
- El adaptador de DeepSeek envuelve un fallo de transporte anterior a la respuesta en `LlmError('TRANSPORT')` nombrando el `baseURL` configurado y encadenando el rechazo original como `cause`. Una solicitud abortada se convierte en `LlmError('ABORTED')`; como la señal del turno ya está abortada, el agent loop (bucle del agente) sigue clasificando el turno como cancelación en lugar de recuperación.
- Cada frontera de diagnóstico renderiza a través de `errorChain` en lugar de `error.message`/`String(error)`: el mensaje de error perdurable de `turn/end` del agent-loop (su `errorData`), sus advertencias de logger, el aviso `agent/error` de la TUI y su línea de fallo de arranque, y las líneas de log de fallo de arranque de `dsh-stdio`. El evento en vivo `agent/error` y `SettleReason` conservan el valor lanzado como `unknown`; cada Consumer de diagnóstico lo renderiza en lugar de que el loop lo envuelva en otro error. Las copias de `renderThrown` por paquete en `dsh-agent-loop`, `dsh-stdio` y `dsh-tui` se eliminan en favor del único renderizador compartido.
- `dsh-stdio` renderiza las razones de `turn/end` fallidas: `[turn failed <code>] <message>`, `[turn aborted] <reason>`, `[turn rejected] <reason>`, `[turn interrupted by a previous process exit]` y el aviso de límite de tokens de salida. Los tipos extendidos por merge desconocidos caen como finales de turno ordinarios.

`errorChain` vive en `dsh-llm` junto a `HarnessError` por la misma razón que la clase base: es el paquete hoja que todo Consumer ya importa, así que compartirlo no cuesta ninguna arista de dependencia nueva.

## Alternativas consideradas

**Renderizar la cadena dentro del constructor de cada error (incorporar la causa al `message`).** Rechazada: renderiza dos veces una vez que los consumidores también recorren `cause` (el primer borrador de la corrección del adaptador produjo `… fetch failed: bad port: fetch failed: bad port`), y destruye la cadena estructurada para los consumidores que quieren enrutar sobre el error interno.

**Solo un exportador de logger consciente de `cause`.** Rechazada: la razón perdurable de `turn/end` y el aviso de la TUI no son líneas de logger; el mensaje enmascarado persistiría en el log de la sesión — el único registro perdurable de un fallo dentro del turno — y en la UI principal.

**Mejoras de `renderThrown` por paquete.** Rechazadas: tres paquetes ya llevaban copias privadas casi idénticas; mejorar cada una por separado afianza la duplicación que elimina el renderizador compartido.

## Consecuencias

- Un fallo de transporte ahora se lee como `DeepSeek API request to <baseURL> failed: fetch failed: connect ECONNREFUSED …` en el aviso de la TUI, en el transcript (transcripción) de readline y en el log de sesión persistido, a costa de cadenas de diagnóstico más largas.
- Los mensajes de error perdurables de `turn/end` incluyen el detalle de la causa. Los fixtures (datos de prueba) de instantáneas existentes se reproducen byte a byte porque sus errores programados no llevan `cause` (para esos errores `errorChain(err)` es igual a `err.message`); solo cambiaron las cadenas de expectativa de las pruebas unitarias. Un fixture grabado de un fallo de transporte real llevaría la cadena.
- `errorChain` renderiza `message` sin el nombre de la clase (`String(error)` renderizaba `Error: <message>`), así que un `TypeError` desnudo en una línea de log pierde su etiqueta de tipo salvo que su mensaje esté vacío (entonces el nombre es el respaldo). El detalle de la cadena se juzgó más valioso que el nombre de la clase en estas fronteras de diagnóstico.
- La salida de `dsh-stdio` para turnos fallidos ya no es silenciosa; los consumidores con pipe que parseaban el transcript ven nuevas líneas `[turn …]`.
- Las copias restantes de `renderThrown` en `dsh-subagent`, `dsh-workflow`, `dsh-skill` y `dsh-workflow-worker-thread` siguen renderizando sin la cadena; envuelven errores locales al paquete que llevan sus propios mensajes, y pueden adoptar `errorChain` cuando sus diagnósticos se demuestren insuficientes.
