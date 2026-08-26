# Agent Note: Microkernel — extension via Cordis event taxonomy, one concrete loop

[English](2026-06-11-microkernel-event-taxonomy.md) | Español

Status: implemented

## Problem

El principio del producto es «todo es un plugin»: hooks, /goal, /loop, flujos de trabajo dinámicos, compactación, sandboxing, permisos, UI, persistencia, MCP, skills deben poder escribirse como plugins sin modificar el núcleo.

## Decision

Taxonomía de eventos Cordis pura. Los puntos de extensión del bucle son eventos tipados con modos de despacho deliberados:

- **waterfall** (middleware around) donde los plugins transforman, cortocircuito, recuperan o envuelven: `agent/pre-step`, `agent/request`, `agent/request-error`, `tools/pre-execute`, `tools/execute`, `tools/post-execute`, `llm/stream`, `system-prompt/assemble`.
- **serial** (esperado en orden de listener) para puntos de control ordenados como `agent/turn-stopping`.
- **parallel** (fan-out esperado) donde cada listener debe tener una oportunidad independiente: el punto de control de durabilidad `session/flush`.
- **emit** (fuego-y-olvido síncrono) para notificaciones: transiciones de bandeja de entrada, ciclo de vida, errores y la observación inmutable `tools/result` contenida. Los eventos de sesión durables poseen los límites de turno y paso.

El vocabulario de eventos vive en paquetes de contrato (`dsh-agent` declara los eventos `agent/*`); `@deepseek-ai/dsh-agent-loop` es el único plugin de bucle concreto y es intercambiable en sí mismo — nada fuera de él puede depender de él.

## Alternatives considered

**Una pila de middleware de propósito específico (estilo koa-compose)** y **una máquina de estado de fase explícita en la que insertan los plugins** — ambos reimplementarían la semántica de despacho, eliminación y recarga que el sistema de eventos nativo de Cordis ya proporciona; como efectos de Cordis, los listeners obtienen HMR y eliminación gratis.

## Consequences

- Cada funcionalidad del MVP se mapea a un listener (el [mapa funcionalidad → mecanismo](../../../../docs/cookbook/extension-cookbook.md#the-feature--mechanism-map) es la obligación de prueba, mantenido actualizado).
- HMR y eliminación son gratuitos: los listeners y los registros son efectos de Cordis.
- La semántica waterfall (llama `next()` o cortocircuita) no es obvia y debe enseñarse — documentado en AGENTS.md y cubierto por pruebas de composición.
- El bucle debe ser defensivo: las excepciones de plugin se contienen a nivel de turno, la dirección desde cualquier punto de extensión nunca queda abandonada (probado por regresión).