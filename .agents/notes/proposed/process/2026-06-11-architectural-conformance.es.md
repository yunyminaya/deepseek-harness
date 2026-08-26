# Agent Note: Conformidad arquitectónica — reglas de dependencias y el kit de adaptadores

Status: proposed

[English](2026-06-11-architectural-conformance.md) | Español

## Problema

Dos garantías arquitectónicas viven hoy solo en prosa: (1) nada depende del paquete de bucle concreto ([la promesa del micrornúcleo](../../implemented/architecture/2026-06-11-microkernel-event-taxonomy.es.md)) y (2) cada LlmAdapter habla el protocolo de chunks correctamente. Ambas deberían ser mecánicas ([el principio de quality gates](../../implemented/process/2026-06-11-quality-gates.es.md)).

## Propuesta

**dependency-cruiser** con reglas:

- `packages/*` (salvo las propias pruebas de agent-loop y examples/) no debe importar `@deepseek-ai/dsh-agent-loop`.
- Sin imports profundos entre paquetes (rutas `@deepseek-ai/dsh-*/src/...`) — solo puntos de entrada públicos.
- Sin ciclos de imports en todo packages/.
- `vendor/*` no debe importar desde `packages/*`.
- Estratificación: dsh-llm no importa nada de otros paquetes dsh; dsh-session solo dsh-llm; etc. (la tabla de dependencias de packages/README.md, aplicada).

**Kit de conformidad de adaptadores** en dsh-llm (`@deepseek-ai/dsh-llm/conformance`): una suite vitest reutilizable parametrizada por una fábrica de adaptadores, que afirma el contrato del protocolo de chunks — monotonía de índice por bloque, sin deltas tras `block-end` para un índice, exactamente un `finish`, el usage a lo sumo una vez, cada `tool-call-delta` lleva el id de llamada, la anulación honrada con prontitud. Ejecutarlo contra los mocks ahora; el adaptador de DeepSeek V4 lo hereda desde el primer día. Opcionalmente un wrapper `strictAdapter()` en modo dev que aplique lo mismo en runtime tras una bandera de depuración (en pareja con [los invariantes de modo desarrollo](../../implemented/architecture/2026-06-11-dev-invariants-over-deep-readonly.es.md)).

## Plan

Primero la configuración de dependency-cruiser + el paso de CI (una hora de trabajo, garantía permanente); el kit de conformidad aterriza con su primera prueba de consumidor contra MockAdapter, y es prerrequisito de la fase del adaptador V4.

## Criterios de aceptación

- dependency-cruiser corre en CI con las familias de reglas anteriores; un import que las viole falla el build.
- El kit de conformidad corre contra el adaptador mock y ambos adaptadores entregados, y un paquete de adaptador nuevo hereda la suite invocándola con su fábrica.

## Riesgos

Mantenimiento de las reglas de dep-cruiser a medida que se añaden paquetes — mantener las reglas basadas en patrones (`dsh-*`) en lugar de enumeradas.

<!-- agent-note-format: alternatives-not-recorded (pre-format Agent Note) -->
