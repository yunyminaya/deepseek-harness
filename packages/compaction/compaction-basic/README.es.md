# @deepseek-ai/dsh-compaction-basic

[English](README.md) | Español

El **backend de compactación básico**: un `BasicCompactionEngine` que implementa la Service Definition de `@deepseek-ai/dsh-compaction` con presión reutilizable de `ctx.tokenMeter`, retención de presupuesto de tokens y resumen como llamada directa de un solo uso a `ctx.llm.stream()` que reproduce el prefijo de la conversación para reutilizar la caché KV del provider (interceptable en `llm/stream`).

Este paquete posee el rol de Service Provider de la capacidad de compactación — ver el [paquete de Service Definition](../compaction/README.es.md) para su contrato y la [Agent Note de capability-seam](../../../.agents/notes/implemented/feature/2026-06-18-compaction-capability-seam.es.md) para el diseño.

## Qué posee

Este backend posee la política de compactación:

- **Medición** — el singleton `ctx.tokenMeter` fija el precio del último envelope canónico registrado y de la superficie actual en una revisión de log consumida. La presión en el límite de paso incluye por tanto el system prompt real, las herramientas, el enrutado, la completación del assistant, los resultados de herramienta, el contexto almacenado en buffer y el steering (direccionamiento).
- **Política enrutada** — la presión proactiva resuelve la capacidad desde el adaptador que posee la ruta durable provider/model más reciente, y luego escala la política predeterminada más una anulación opcional de objetivo exacto hasta presupuestos de tokens concretos. El descubrimiento de modelos sigue siendo consultivo y no se consulta.
- **Poda sin modelo** — cuando la presión o el desbordamiento canónico califican, el servicio opcional [`ctx.toolResultPruner`](../compaction-tool-result-pruner/README.es.md) reescribe los resultados de herramienta sobredimensionados antes de la selección de rango. Compact-basic vuelve a medir a través de `ctx.tokenMeter`, omite el resumen cuando la presión se vuelve segura y, en caso contrario, resume la superficie podada. Las comprobaciones de paso por debajo de la presión nunca podan.
- **Retención** — compacta las unidades de superficie completas más antiguas conservando una cola reciente y cortes equilibrados de llamada de herramienta/resultado a través de los [helpers de límite de `dsh-compaction`](../compaction/README.es.md#tool-pairing-boundaries). Los límites de turno no protegen los pasos antiguos dentro de un turno desbocado. Una cola indivisible abierta declina hasta que se cierra. El pruner opcional puede reparar una unidad de herramienta cerrada sobredimensionada cuando su resultado con texto es el grueso removible; las unidades indivisibles que no son de herramienta y los restos de herramienta no podables quedan fuera del alcance.
- **Convergencia** — reintenta la compactación del checkpoint de cabeza hasta `compactionRetries`; rechaza un resumen que no reduzca su fuente y lanza si los reintentos no pueden volver por debajo del umbral.
- **Resumen** — una llamada directa `llm/stream` usa el par provider/model y el tope configurados, con respaldo al objetivo de solicitud registrado más reciente y después al objetivo del agent, sin ejecutar el punto de extensión `agent/request` solo de bucle. La llamada reproduce verbatim el system prompt propio de la conversación, las herramientas y los mensajes de la región ensombrecida, incluidas las referencias a imágenes, y añade la instrucción de compactación como mensaje final de usuario, de modo que reutiliza la caché cálida de prefijo del provider en lugar de invalidarla. El adaptador seleccionado debe resolver o rechazar explícitamente esas imágenes. Establece `GenerateOptions.purpose` a `compaction`, que los adaptadores pueden reenviar como atribución de solicitud (el adaptador de DeepSeek envía `x-deepseek-harness-compact: 1`) sin tocar el cuerpo visible para el modelo. Solo el texto devuelto entra en el checkpoint, excluyendo el razonamiento y las llamadas de herramienta que filtrarían razonamiento privado o crearían una llamada huérfana; la salida de imagen falla con `UNSUPPORTED_CONTENT` en lugar de desaparecer.
- **Enmarcado** — el mensaje de usuario de reemplazo marca el contexto de checkpoint establecido con etiquetas `<compacted-summary>`. El resumen crudo permanece en el evento `compaction/summary`, y los ciclos automáticos posteriores fusionan el checkpoint previo.
- **Ciclo de vida** — todos los puntos de entrada comparten una transacción de región con el bracket primero. Valida el rango y el bloqueo vivo, añade `compaction/start` de forma síncrona, prepara y espera el resumen, revalida, añade `compaction/summary` más el reemplazo, y hace exactamente un intento de cierre. Las llamadas automáticas y de región explícita requieren un dueño de turno abierto numérico y estabilidad de la superficie completa; el listener serial `agent/pre-step` comprueba la presión antes de la derivación de la solicitud, mientras que el desbordamiento canónico del provider entra por `agent/request-error` y autoriza el reintento solo después de progreso durable de la superficie. `compactNow()` reserva la admisión de reposo, usa `turn: null`, acepta contexto de solo añadido fuera de su tramo seleccionado, vacía cada intento cerrado y libera la admisión en `finally`.
- **Recuperación de desbordamiento** — el desbordamiento confirmado por el provider no necesita metadatos de capacidad: se salta la presión y la retención normales, poda y luego intenta una reducción máxima equilibrada de cabeza dejando la unidad indivisible más nueva. El reintento se autoriza siempre que `surface.replaceGeneration` avance, incluso cuando la poda aterriza antes de que el trabajo de resumen posterior lance. Ningún reemplazo, un tope específico de objetivo agotado, la cancelación o un error desconocido/no canónico preservan el fallo original del provider.
- **Manejo de fallos** — un `compaction/start` vivo no emparejado es el bloqueo durable. Un marcador no emparejado anterior a un `session/end-seed` más reciente es evidencia obsoleta de un ciclo de vida previo y no bloquea; uno posterior a ese límite informa `busy`. Los fallos de resumen y de tramo cambiado cierran con un error y dejan intacta la superficie de la conversación, aunque el intento permanece en el log. Un cierre fallido deja deliberadamente un huérfano bloqueante. Los fallos operativos de presión avisan y continúan, mientras que el fallo de recuperación de desbordamiento preserva el error original del provider solo cuando ningún reemplazo anterior avanzó la superficie. La cancelación sigue siendo autoritativa después de la limpieza y la durabilidad.

El método protegido `summarize()` es el único hook de subclase. Una subclase de resumidor por plantilla o remoto puede anularlo mientras la presión, la retención, los eventos fuente citados, la validación de reducción y la contabilidad de tokens ensombrecidos permanecen en `ctx.tokenMeter`. El hook devuelve el resumen seguro más la salida completa del provider, el envelope de llamada y el uso cuando están disponibles (`{ summary, rawOutput?, llmStreamCall?, provider, model, maxTokens?, usage? }`); `llmStreamCall: true` significa que producir ese resultado consumió exactamente una llamada a través del `ctx.llm.stream()` de este contexto y requiere `rawOutput` completo, mientras que un `rawOutput` sin marcar no identifica la vía de llamada. La transacción conserva esos campos en `compaction/summary`.

## Config (`BasicCompactionConfig`)

Cada ajuste es opcional. Los campos de política de nivel superior son valores predeterminados para cada modelo enrutado; `modelPolicies` aplica anulaciones parciales a pares exactos provider/model. En el momento de la presión, compaction-basic pide al adaptador LLM propietario la capacidad de contexto de esa ruta y resuelve presupuestos absolutos. Las claves no reconocidas, los objetivos duplicados, las formas de retención mutuamente excluyentes y un `retainRatio` fusionado que no esté por debajo de `thresholdRatio` fallan la carga del plugin. Un presupuesto absoluto `retainTokens` que no esté por debajo de su umbral escalado falla en el primer objetivo resoluble porque esa comparación requiere capacidad del modelo.

| Clave | Obligatorio | Significado |
|---|---|---|
| `thresholdRatio` | no (valor predeterminado `0.8`) | Compacta en `floor(routedContextWindow × ratio)`. |
| `retainRatio` | no (valor predeterminado `0.16`) | Presupuesto de superficie reciente conservado verbatim como fracción de la ventana de contexto enrutada; mutuamente excluyente con `retainTokens`. |
| `retainTokens` | no | Presupuesto absoluto de superficie reciente conservado verbatim; mutuamente excluyente con `retainRatio` y debe estar por debajo del umbral resuelto. |
| `summarizationProvider` | no (valor predeterminado `''`) | Se fija junto con `summarizationModel`; un par vacío resuelve el objetivo de solicitud registrado más reciente y después el par de `AgentOptions`. |
| `summarizationModel` | no (valor predeterminado `''`) | Se fija junto con `summarizationProvider`; un par vacío resuelve el objetivo de solicitud registrado más reciente y después el par de `AgentOptions`. |
| `maxTokens` | no (valor predeterminado `8192`) | Tope de generación del provider para la llamada de resumen; puede incluir tokens de razonamiento. |
| `compactionRetries` | no (valor predeterminado `1`) | Intentos extra después del primero cuando la presión sigue por encima del umbral. |
| `maxOverflowRetries` | no (valor predeterminado `1`) | Máximo de reintentos después del desbordamiento canónico de la ventana de contexto; `0` solo desactiva la recuperación. |
| `modelPolicies` | no (valor predeterminado `[]`) | Anulaciones exactas `{ provider, model, ...partialPolicy }`; el emparejamiento usa ambos campos y no depende de `listModels()`. |
| `auto` | no (valor predeterminado `true`) | Registra los listeners de presión en el límite de paso y de recuperación de desbordamiento. Pon `false` para solo manual. |

Cada entrada de `modelPolicies` acepta los campos de política anteriores excepto `auto` y el propio `modelPolicies`. Si una entrada suministra cualquiera de los campos de retención, reemplaza la elección de retención de la política predeterminada; en caso contrario, la retención se hereda. El provider/model de resumen sigue siendo un par dentro de cada entrada.

Un adaptador puede no devolver capacidad para una ruta dinámica válida, y la capacidad resuelta puede exponer un presupuesto de retención absoluto inválido. Las comprobaciones manuales de presión lanzan entonces un error de configuración específico del objetivo; el listener automático avisa una vez para ese objetivo exacto y continúa con el historial completo. Los fallos operativos no relacionados siguen siendo visibles de forma independiente. El desbordamiento canónico del provider aún intenta la recuperación porque el provider ya ha establecido que la compactación es necesaria.

## Uso

`BasicCompactionEngine` requiere `ctx.llm`, `ctx.tokenMeter` y `ctx.sessions`. La composición siguiente recibe `ctx.llm` de su host e instala los otros dos servicios:

```ts
import type { Context } from '@deepseek-ai/cordis'
import { BasicCompactionEngine } from '@deepseek-ai/dsh-compaction-basic'
import SessionStore from '@deepseek-ai/dsh-session'
import TokenMeter from '@deepseek-ai/dsh-token-meter'

export const name = 'compaction-basic'
export const inject = ['llm']

export function apply(ctx: Context): void {
  ctx.plugin(SessionStore)
  ctx.plugin(TokenMeter)
  ctx.plugin(BasicCompactionEngine)
}
```

Cargar el plugin registra `ctx.compaction`. Añade [`dsh-compaction-tool-result-pruner`](../compaction-tool-result-pruner/README.es.md) como hermano antes de este plugin para habilitar la pasada opcional sin modelo. Con `auto: true` (el valor predeterminado) compacta automáticamente bajo presión de tokens. El hermano [`dsh-command-compact`](../command-compact/README.es.md) llama a `ctx.compaction.compactNow(...)`; los llamadores programáticos también pueden usar directamente cualquier operación del seam.

Por ejemplo, el mismo plugin de compactación puede servir de forma segura a modelos con capacidades distintas y una política específica de objetivo:

```yaml
- name: '@deepseek-ai/dsh-compaction-basic'
  config:
    thresholdRatio: 0.8
    retainRatio: 0.16
    modelPolicies:
      - provider: local
        model: small-context
        thresholdRatio: 0.7
        retainTokens: 2048
```

## Experiencia del modelo

### Historial de conversación

#### Qué ve el modelo

Después de que un paso con éxito cruza el umbral, los resultados de herramienta sobredimensionados se reescriben primero cuando el pruner opcional está cargado. Si el resumen sigue siendo necesario, la siguiente solicitud recibe el preámbulo de checkpoint de abajo, una línea en blanco, `<compacted-summary>`, el resumen dependiente de los datos y `</compacted-summary>`. La recuperación de desbordamiento reconstruye el reintento inmediato a partir de cualquier reemplazo que avanzó la superficie. Un checkpoint reemplaza el rango anterior seleccionado y le siguen las unidades recientes retenidas.

##### Preámbulo de checkpoint de conversación

```markdown
This is an automatically generated checkpoint condensing an earlier span of the conversation to free up context. Treat the captured context as established background and build on it without restating it. Continue the task directly from the messages that follow, without acknowledging this checkpoint.
```

#### Efecto en tokens

La poda sin modelo puede evitar por completo la llamada auxiliar; en caso contrario, reduce el transcript de esa llamada antes de que el resumen reemplace un rango anterior. El reemplazo reduce el historial de entrada futuro en lugar de añadir una segunda copia. Un resumen permanece hasta que una compactación posterior lo reemplaza, mientras que una unidad indivisible que no es de herramienta aún puede exceder el presupuesto.

#### Efecto en la caché KV

Reemplazo en lugar de solo añadido. Cada checkpoint invalida la reutilización desde el primer token de historial reemplazado; el prefijo de solicitud sin cambios anterior a ese rango sigue siendo reutilizable.

### Solicitud auxiliar de resumen

#### Qué ve el modelo

El modelo de resumen recibe la conversación reproducida verbatim — el mismo system prompt, los mismos schemas de herramienta y los mismos mensajes que envió la última solicitud enrutada para la región ensombrecida — seguidos de un mensaje final de usuario: la instrucción de compactación de abajo. El modelo de conversación nunca ve esta solicitud privada ni su razonamiento; solo se almacena el texto devuelto.

##### Instrucción de compactación (mensaje final de usuario)

```markdown
You are now acting as a compaction engine for this AI coding assistant. Condense the conversation ABOVE into a structured checkpoint that lets another model resume the work with no loss of essential context.

Output EXACTLY the Markdown structure below: keep every section, in order. Use terse bullets, not prose paragraphs. Write "(none)" for an empty section — never drop a section.

## Primary Request and Intent
- [the user's original and evolving goals; quote verbatim where the exact wording matters]

## Key Technical Concepts
- [technologies, frameworks, patterns, and conventions in play]

## Files and Code
- [exact path: why it matters, key changes or snippets]

## Errors and Fixes
- [error: how it was resolved, plus any related user feedback]

## Pending Jobs
- [explicitly requested work not yet completed]

## Current Work
- [precisely what was in progress at this checkpoint]

## Next Step
- [the single next action, directly in line with the most recent request, or "(none)"]

## Critical Context
- [decisions and their rationale, constraints, user preferences, open questions, data needed to continue]

Rules:
- Write concise English engineering prose. Preserve exact file paths, commands, error strings, identifiers, numeric values, function signatures, and syntax fragments.
- Capture user feedback and explicit instructions faithfully, especially corrections.
- Do NOT mention this summarization request or that the context was compacted.
- Output only the checkpoint text: do not call any tool or take any other action.
- If the conversation already contains a <compacted-summary> block, it is a PRIOR checkpoint. Do not copy it forward verbatim: preserve still-true facts, drop stale ones, and merge newer information into a single consolidated summary under the same structure.
```

#### Efecto en tokens

Esta es una llamada de modelo separada: el prefijo de conversación reproducido más la instrucción fija como entrada, con salida limitada por `maxTokens`. Los reintentos de convergencia pueden pagar este coste más de una vez.

#### Efecto en la caché KV

El system prompt reproducido, las herramientas y los mensajes de la región ensombrecida coinciden byte a byte con la última solicitud enrutada de la conversación, de modo que la caché cálida de prefijo del provider se reutiliza hasta la instrucción final; solo esa instrucción, y la salida del resumen, no están en caché. Enrutar el resumidor a otro provider/model, o compactar un rango que no es de cabeza, renuncia a esta reutilización.

## Limitaciones conocidas y trabajo pendiente

- **La precisión del meter sigue la heurística fija** — la ausencia de uso reutilizable del provider cae a recuento de caracteres más overhead estructural en lugar de tokenización exacta.
- **La clasificación de desbordamiento la mantiene el adaptador** — la redacción del provider puede cambiar; ambos adaptadores de DeepSeek normalizan los fallos de límite de contexto reconocidos actualmente a `CONTEXT_WINDOW_EXCEEDED`.
- **Parte del desbordamiento de unidades indivisibles y solo de envelope queda fuera de la compactación de superficie** — la recuperación no puede reducir system/tools/prefix, dividir un nodo indivisible que no es de herramienta ni reparar una unidad de herramienta cuyo resto no podable aún excede la ventana. El pruner opcional puede reducir el grueso con texto de un resultado de herramienta dentro de un par por lo demás indivisible.
- **`compactRegion` requiere un turno abierto** — una llamada manual en una sesión totalmente cerrada lanza («no open turn») en lugar de compactar.
- **El fallo de resumen conserva la superficie durable más reciente** — antes de cualquier reemplazo, la vía automática registra una advertencia y continúa con el historial completo por encima del presupuesto. Si la poda ya aterrizó, un fallo de resumen posterior continúa desde esa superficie podada durable. La truncación del resumen en `maxTokens`, que los tokens de razonamiento ocultos pueden consumir, sigue la misma regla.
