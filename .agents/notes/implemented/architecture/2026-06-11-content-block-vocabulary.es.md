# Agent Note: Provider-neutral content-block vocabulary owned by dsh-llm

Status: implemented

[English](2026-06-11-content-block-vocabulary.md) | Español

## Problem

El harness necesita un lenguaje interno para mensajes que el loop, el registro de sesión y todos los plugins hablen.

## Decision

Apropiar el vocabulario: los mensajes son arrays de bloques de contenido tipados (`text`, `reasoning`, `tool-call`, `tool-result`), con la unión derivada del mapa extensible por fusión `ContentBlockMap` para que los plugins agreguen tipos de bloques mediante fusión de declaraciones. El mismo patrón de mapa extensible por fusión tipifica cada campo "stringly" (`MessageSource`, `FinishReason`, `TurnTrigger`, `TurnEndReason`). Streaming es un protocolo de chunk en bruto; `BlockAssembler` es la única implementación de ensamblaje compartida. Los adaptadores traducen a formatos de cable de provider — el costo de mapeo vive en los adaptadores, donde corresponde.

La inyección de contexto en sesión (`context/message`) y el direccionamiento a mitad de turno originalmente se renderizaban como envoltorios con rol de usuario etiquetados (el patrón de recordatorio del sistema) en lugar de un nuevo rol, por lo que los adaptadores no cargan ningún peso. Ambos ahora proyectan como contenido de usuario plano sin envoltorio; ver [el Agent Note de envoltorio de contenido inyectado](../simplification/2026-07-20-unwrap-injected-content-envelopes.es.md). La validación de adaptador en vivo confirma este renderizado para el comportamiento actual de DeepSeek; una falta de coincidencia específica de un provider futuro pertenece a ese adaptador en lugar de un rol canónico nuevo.

## Alternatives considered

- **Reflejar la forma de chat-completions de DeepSeek/OpenAI** — costo de mapeo cero para el primer provider, pero incómodo para contenido rico (razonamiento, resultados de herramientas como bloques estructurados).
- **Adoptar la estructura de bloques Messages de Anthropic textualmente** — probada en batalla, pero los tipos canónicos reflejarían una API de terceros que el harness no apunta primero.

## Consequences

- Reasoning tiene un hogar central sin formas específicas de provider.
- Los bloques multimodales retornan solo con soporte coordinado de adaptador, UI y compactación; ver [el Agent Note de eliminación de imagen](../../archived/simplification/2026-07-04-drop-image-content-block.es.md).
- Las sugerencias de caché y el prefill de asistente permanecen ausentes hasta que un adaptador enviado pueda honrarlos; ver las [variantes sin productor](../../archived/simplification/2026-07-04-prune-producerless-vocabulary-variants.es.md) y los [controles de solicitud inertes](../../archived/simplification/2026-07-04-drop-inert-request-knobs.es.md) Agent Notes.
- Cada adaptador paga un costo de traducción; los primeros adaptadores reales han validado desde entonces el protocolo de streaming, y los nuevos adaptadores deben seguir probando su mapeo específico de provider en pruebas locales de adaptador.
- Los IDs que cruzan límites de paquete están marcados (`CallId`, el `SessionId` compartido de agente/sesión) — tipificación nominal a costo cero en tiempo de ejecución.