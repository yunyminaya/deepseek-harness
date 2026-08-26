# Agent Note: Schemas en runtime para el vocabulario de eventos (Zod vs el patrón de mapa merge-extensible)

Status: proposed

[English](2026-06-16-typed-event-schemas.md) | Español

## Problema

El harness modela su vocabulario central — bloques de contenido, fuentes de mensaje, razones de finish, disparadores de turno, razones de fin de turno y eventos de sesión — como **mapas merge-extensibles**: un `interface` de TypeScript (p. ej. `SessionEventMap`, `ContentBlockMap`) que los plugins aumentan vía declaración-merging, con la unión pública derivada como `Map[keyof Map]`. Este es el patrón de extensión universal del repo, documentado en [docs/architecture.md](../../../../docs/architecture.es.md) («El mismo patrón de mapa merge-extensible se usa para `MessageSource`, `FinishReason`, `TurnTrigger` y `TurnEndReason`») y del que dependen el DSL `InferArgs` de `defineTool` y la convención de exhaustividad `assertNever`.

El patrón es **solo de compilación**. Los tipos desaparecen en runtime: no hay ningún objeto schema contra el que validar un valor entrante, con el que parsear entrada no confiable o que enumerar en runtime. El [contrato de persistencia de sesión](../../implemented/architecture/2026-06-14-session-persistence.es.md) expone dos consecuencias:

1. **La persistencia trata `event.data` como JSON opaco.** Los backends JSONL/SQLite hacen `JSON.stringify`/`JSON.parse` de cada evento verbatim; la única guardia de runtime es `isJsonValue` (serializabilidad de ida y vuelta — rechaza BigInt, funciones, ciclos, números no finitos, …), NO la validación estructural. Un dato de evento corrupto-pero-todavía-JSON (tipos de campo equivocados, campos faltantes) hace ida y vuelta en silencio y solo se atrapa después, si acaso, por el `switch` de un consumidor.
2. **Sin contrato de runtime para las variantes añadidas por plugins.** Un plugin que hace declaración-merging de una nueva clave de `SessionEventMap` obtiene tipado de compilación para su propio código, pero nada valida que los valores que produce coincidan con la forma que declaró — en el productor, en el límite de persistencia o al recargar.

Esto plantea si el vocabulario de eventos debería mudarse a **Zod** u otra biblioteca de schemas de runtime para que los límites durables y de plugins tengan schemas de runtime en lugar de tipos borrados.

## Por qué esto no es un cambio de persistencia

Es tentador leer «usa Zod para la serialización» como un cambio local a `dsh-session-persistence-jsonl/src/format.ts`. No lo es, por una razón estructural: **un plugin no puede hacer declaración-merging de un schema de Zod.** El declaración-merging es un mecanismo de compilación de TypeScript; un schema de Zod es un valor de runtime. Para validar eventos con Zod necesitas un **registro de runtime** al que cada paquete productor de eventos contribuya su schema (p. ej. `ctx.sessionEvents.register('compaction/marker', z.object({…}))`), y del que lea cada consumidor. Ese registro — no el backend de persistencia — se convierte en la fuente de verdad del vocabulario, reemplazando al interface merge-extensible.

Así que la propuesta real es: **reemplazar el patrón de mapa merge-extensible de compilación por un registro de schemas de runtime, en todo el repo.** Eso es un rediseño del vocabulario central.

## Radio de explosión (medido)

Una migración de la API de eventos/vocabulario a schemas de runtime toca, como mínimo:

- **Seis mapas merge-extensibles** (~370 LOC de tipos centrales): `ContentBlockMap`, `MessageSourceMap`, `FinishReasonMap` (en `dsh-llm`); `TurnTriggerMap`, `TurnEndReasonMap`, `SessionEventMap` (en `dsh-session`).
- **~10 sitios de aumento `declare module`** en `dsh-agent`, `dsh-agent-loop`, `dsh-shell`, `dsh-llm`, `dsh-session`, `dsh-session-persistence`, `dsh-system-prompt`, `dsh-tools` — cada uno pasaría de declaración-merging a una llamada `register()` de runtime.
- **Los productores de eventos** — 16 sitios de llamada `session.append(...)` en el loop — sin cambio de forma pero ahora validados en el límite.
- **~7 consumidores switch** que ramifican sobre estas uniones: `deriveMessages` y el compañero invariante propiedad del paquete (`dsh-session`), `BlockAssembler` (`dsh-llm`), ambos adaptadores LLM (`dsh-llm-deepseek`, `dsh-llm-pi-ai`) y la capa de schemas de herramienta (`dsh-tools`). La convención de `assertNever`-en-uniones-cerradas vs fall-through-en-uniones-extensibles (una regla de lint documentada) necesitaría replanteo — las variantes de runtime no son estáticamente exhaustivas.
- **El DSL `InferArgs` de `defineTool`** (`dsh-tools`), que deriva tipos de argumento `execute` sin cast de un spec de schema de compilación — el escaparate del enfoque actual.
- **Docs**: architecture.md (el patrón se describe como fundacional), [invariantes de dev-mode](../../implemented/architecture/2026-06-11-dev-invariants-over-deep-readonly.es.md) y cualquier Agent Note que referencie el patrón.

Esto es un rediseño del vocabulario en todo el repositorio, no un detalle de implementación de persistencia.

## Alternativas consideradas

### A. Status quo — tipos merge-extensibles + `isJsonValue` en el límite durable

Conservar el patrón de compilación. La persistencia sigue siendo JSON opaco + guardia de serializabilidad. Los plugins extienden vía declaración-merging; la corrección de la *forma* del evento es responsabilidad del productor y la impone TypeScript en compilación. Los compañeros invariantes propiedad del paquete comprueban relaciones seleccionadas entre registros cuando están habilitados pero no proporcionan schemas generales de forma en runtime.

- **Pros**: cero churn; la extensión de plugins es un `interface` de una línea con inferencia completa de tipos y sin ceremonia de registro en runtime; sin dependencia nueva de runtime; el DSL `defineTool` y la exhaustividad `assertNever` siguen funcionando.
- **Contras**: sin validación estructural de runtime en el límite de persistencia ni en los límites de plugins; un dato malformado-pero-JSON se atrapa tarde.

### B. Solo validación de cabecera/forma cerrada (schemastery), los eventos siguen opacos

Endurecer solo las formas genuinamente cerradas que ya tienen guardias de tipos hechas a mano — p. ej. la guardia `HeaderLine` del JSONL (`isHeaderLine`) — usando **schemastery** (la biblioteca de schemas existente del repo, ya usada para cada `static Config` de plugin). Dejar la unión de eventos merge-extensible como está.

- **Pros**: pequeña, encaja en la convención existente (schemastery, no una lib nueva); reemplaza las guardias hechas a mano de formas cerradas por schemas declarativos; sin rediseño de core.
- **Contras**: no aborda la validación de datos de eventos; solo mejoran los registros de metadatos fijos.

### C. Registro de schemas de runtime para todo el vocabulario (Zod o schemastery)

Reemplazar los mapas merge-extensibles por un registro de runtime al que contribuyan los productores y contra el que validen los caminos de persistencia/consumidor.

- **Pros**: validación real de runtime en el límite durable y en los límites de plugins; una única fuente de verdad; habilita tooling genérico (docs auto-generadas, fuzzing, comprobaciones de formato de cable).
- **Contras**: todo el radio de explosión anterior; **Zod no es actualmente una dependencia directa** (solo una dep transitiva de `@earendil-works/pi-ai`) y la biblioteca de schemas elegida del repo es **schemastery** — adoptar Zod ampliamente es en sí una decisión de dependencias; la ergonomía del declaración-merging (extensión de plugin de una línea, inferencia completa) se reemplaza por registro en runtime + cableado manual de tipos; la garantía de exhaustividad `assertNever` se debilita (las variantes de runtime no son estáticamente exhaustivas).

## Propuesta

Diferir. Si se quiere validación de runtime en el límite durable, la **Opción B** (schemastery sobre las formas cerradas de cabecera y metadatos) es el paso proporcionado dentro de la convención existente. La **Opción C** es una decisión de arquitectura que requiere su propio Agent Note de implementación, incluida una elección entre Zod y schemastery.

## Criterios de aceptación

- La Opción C procede solo a través de su propio Agent Note de implementación, nunca como efecto secundario de persistencia.
- Si se adopta la Opción B, las formas cerradas de cabecera/metadatos (la guardia `isHeaderLine` del JSONL y afines) validan a través de schemastery en lugar de guardias hechas a mano, con los mapas merge-extensibles intactos.

## Riesgos

- El diferimiento deja el `data` de eventos estructuralmente sin validar en el límite durable: un dato malformado-pero-JSON se atrapa tarde, por el `switch` de un consumidor — el costo del status quo, aceptado deliberadamente.
- Si la Opción C se adopta alguna vez, la pérdida ergonómica es real: el declaración-merging de una línea se convierte en registro de runtime más cableado manual de tipos, y la garantía de exhaustividad estática de `assertNever` se debilita.

## Preguntas abiertas

- Si se adopta un registro, ¿la biblioteca es **schemastery** (ya en el árbol, ya la lib de schemas de config) o **Zod** (ecosistema más rico, hoy solo transitiva)? Adoptar dos bibliotecas de schemas es en sí un costo.
- ¿Puede un híbrido conservar la inferencia de compilación (para que sobrevivan `defineTool` y la DX de plugins) mientras añade un schema de runtime *opcional* por variante, validado solo en el límite de persistencia/cable en lugar de en cada append en proceso?
- ¿Cubre ya el servicio `ctx.invariants` suficiente de la brecha de forma en runtime cuando está habilitado como para que la validación de límite solo sea necesaria para entrada genuinamente no confiable (recarga de un log modificado externamente)?
