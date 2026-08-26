# Agent Note: Semántica de dominios de eventos — la sesión es el registro de hechos, el agent es el canal de eventos en vivo

Status: implemented

[English](2026-06-30-event-domain-semantics.md) | Español

## Problema

El harness extiende el agent loop (bucle del agente) a través de una taxonomía de eventos de Cordis (véase [el Agent Note de taxonomía de eventos del microkernel](2026-06-11-microkernel-event-taxonomy.es.md)). A medida que esa taxonomía creció, la línea entre los tres dominios de eventos se desdibujó:

- `session/*` transporta el registro duradero y event-sourced (`SessionEventMap`).
- `agent/*` transporta señales de runtime en vivo que entregan el handle `Agent` a un plugin.
- `tools/*` transporta el registro de herramientas y el pipeline de ejecución.

Dos problemas motivaron fijar la semántica. Primero, varios límites de turno/paso existían TANTO como `SessionEvent` duradero (`turn/start`, `turn/end`, `step/start`, `step/end`) COMO como emisión `agent/*` reflejada (`agent/turn-start`, `agent/turn-end`, `agent/step-start`, `agent/step-end`). Un consumidor tenía dos fuentes de verdad para el mismo hecho, y cada cambio de ciclo de vida tenía que actualizar ambas. Segundo, el subsistema Hooks necesita UNA superficie única, coherente y documentada a la que suscribirse — un autor de plugin (y los puentes de hooks de Claude Code / Codex construidos encima) debe saber, sin leer el loop, si escuchar un evento de sesión o un evento de agent, y por qué.

Este vocabulario es la base de las decisiones de intercepción, del registro duradero `hook/*` y de los puentes de Claude Code y Codex.

## Decisión

**Tres dominios, un trabajo cada uno, con una única regla de frontera.**

- **`session/*` — el registro de HECHOS duradero y reproducible.** Es dueño de `SessionEventMap`; cada entrada es solo JSON (sin objetos en vivo). Una emisión `session/event` por cada append, más el punto de control paralelo de durabilidad `session/flush`. También es el feed de transcript (transcripción) en vivo: un consumidor que quiera renderizar o reaccionar a lo sucedido se suscribe aquí, así que el renderizado en vivo y las proyecciones de reproducción comparten una sola ruta.
- **`agent/*` — la superficie de runtime EN VIVO.** Siempre transporta el `Agent` en vivo. Los waterfalls de intercepción (`agent/pre-step`, `agent/request`, `agent/request-error`) transforman, rechazan o recuperan; el `agent/turn-stopping` en espera observa la frontera de parada; las emisiones transitorias informan ciclo de vida, estado, inserción/reclamación/descartes de la bandeja de entrada y errores. Los LÍMITES de turno y paso NO están aquí — son eventos de sesión duraderos que se leen de `session/event`, igual que el flujo de tokens (`assistant/chunk`) y el steering (direccionamiento) de mitad de turno (un `user/message`).
- **`tools/*` — el registro de herramientas y el pipeline de ejecución.**

**La regla de frontera:** un hecho duradero y reproducible es un `SessionEvent`; una intercepción en vivo o una señal transitoria/de objeto en vivo es un evento de Cordis `agent`/`tools`. Un límite de turno o paso es un hecho duradero, así que vive en el registro de sesión y se lee del feed `session/event` — NO se refleja como emisión `agent/*`.

**Aplicando la regla a los gemelos de frontera:** los cuatro reflejos de frontera — `agent/turn-start`, `agent/turn-end`, `agent/step-start`, `agent/step-end` — se **ELIMINAN**. Ningún consumidor de producción necesita el `Agent` en vivo en una frontera: el puente ACP correlaciona su prompt en vuelo con el par exacto `turn/start`/`turn/end` de `session/event`, y otros consumidores de transcript derivan igualmente las fronteras del flujo duradero. Véase [el Agent Note de eliminación de los eventos espejo de frontera](../simplification/2026-06-20-remove-agent-boundary-mirror-events.es.md), dueño de esa decisión. Eliminar las emisiones también simplifica el `closeStep`/`closeTurn` del loop (un append cada uno, sin emisión pareada).

## Consecuencias

- El loop ya no emite ningún espejo de frontera; `closeStep` añade solo `step/end` y `closeTurn` añade solo `turn/end`. `Session.append` es dueño de la contención de los observers posteriores al commit, así que un observer de frontera que lanza no puede cambiar el resultado del turno ni privar de atención a consumidores posteriores; un fallo de aceptación o de validación interna sigue escapando antes de que la frontera entre en el registro.
- Los tests que observaban fronteras a través de las emisiones eliminadas observan ahora los eventos de sesión duraderos `turn/start`/`turn/end`/`step/start`/`step/end` — el comportamiento que fijan (orden de las fronteras, conteo de pasos) no cambia; solo el feed que leían pasó al canónico. Los tests que ejercitaban un *listener de emisión de frontera de turno que lanza* se eliminaron, porque esa ruta de código ya no existe (no hay emisión desde la que lanzar). Según [AGENTS.md «los tests documentan comportamiento, no verdad dorada»](../../../../AGENTS.md.es.md), el comportamiento y su test se movieron (o murieron) juntos.
- El loop marca el paso como abierto (`stepOpen = true`) solo después de que `append('step/start')` retorne. La validación interna de despacho se ejecuta antes del push al registro y puede rechazar sin abrir un paso; los fallos de los observers de `session/event` posteriores al commit están contenidos dentro de `Session.append`. Por tanto, el marcador representa exactamente la frontera confirmada que debe un `step/end` posterior.
- La realización completa de esto es [el Agent Note de simplificación «Deja de reflejar las fronteras duraderas como eventos de agent»](../simplification/2026-06-20-remove-agent-boundary-mirror-events.es.md): los cuatro reflejos de frontera se eliminan y cada consumidor lee las fronteras de `session/event`. `agent/steering` (que no es un espejo de frontera) quedó fuera del alcance de ese Agent Note y fue eliminado por su propio seguimiento, [Elimina la emisión espejo `agent/steering`](../../archived/simplification/2026-07-04-remove-agent-steering-mirror.md) — reflejaba el `user/message` duradero de steering de mitad de turno.
- La superficie de eventos de Cordis generada (las páginas de `docs/subsystems/`) ya no lista los eventos espejo.

<!-- agent-note-format: alternatives-not-recorded (pre-format Agent Note) -->
