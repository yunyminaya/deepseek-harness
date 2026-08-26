# llm/ — familia de capacidad LLM

[English](README.md) | Español

El seam LLM (modelo de lenguaje de gran tamaño) y sus adaptadores de provider. El paquete `llm` es dueño de los roles de Service Definition y Consumer a la vez: el servicio abstracto, el vocabulario de bloques de contenido y el ensamblador de chunks de stream. Los adaptadores de provider se registran en `ctx.llm`. Todos son paquetes **de producto**.

| Paquete | Rol | Clave de ctx |
|---|---|---|
| [`llm/`](llm/README.es.md) | Servicio LLM y vocabulario de streaming compartido | `ctx.llm` |
| [`token-meter/`](token-meter/README.es.md) | Medición de tokens consciente del replay | `ctx.tokenMeter` |
| [`llm-retry/`](llm-retry/README.es.md) | Política de reintentos con ámbito de provider | escucha `agent/request-error` |
| [`llm-deepseek/`](llm-deepseek/README.es.md) | Adaptador directo de DeepSeek | se registra en `ctx.llm` |
| [`llm-pi-ai/`](llm-pi-ai/README.es.md) | Adaptador pi-ai multiprovider | se registra en `ctx.llm` |

Los adaptadores registran rutas de provider en el seam; el reintento y la medición de tokens siguen siendo Consumers independientes. Los READMEs hijos son dueños de los detalles de enrutado, metadatos, replay y cable del provider; las [decisiones de arquitectura del LLM](../../.agents/notes/implemented/architecture/2026-06-13-twin-llm-adapters.es.md) son dueñas de la justificación.

La referencia del subsistema — mensajes y bloques, la solicitud de modelo, el protocolo `StreamChunk`, el contrato del adaptador — es [docs/subsystems/llm-streaming.md](../../docs/subsystems/llm-streaming.es.md) (medición de tokens: [token-meter.md](../../docs/subsystems/token-meter.es.md)); consulta las Agent Notes de [adaptadores gemelos](../../.agents/notes/implemented/architecture/2026-06-13-twin-llm-adapters.es.md), [medidor de tokens de replay](../../.agents/notes/implemented/architecture/2026-07-15-replay-token-meter-service.es.md) y [contexto de modelo enrutado](../../.agents/notes/implemented/architecture/2026-07-20-routed-model-context-and-compaction-policy.es.md).
