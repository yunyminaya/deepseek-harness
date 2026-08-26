# Agent Note: La llamada de resumen repite el prefijo de la conversación para reutilizar la caché KV

Status: implemented

English | [中文](2026-07-21-compaction-summary-prefix-cache-reuse.zh.md)

## Problema

La compactación automática se dispara a mitad de conversación, justo después de que el bucle ha calentado la caché KV del proveedor con la última petición enrutada (`system` + `tools` + historial derivado). El resumidor por defecto emitía entonces una petición auxiliar *separada* cuyo prefijo no compartía nada con esa petición caliente: un prompt `system` de resumidor a medida seguido del historial más antiguo aplanado a una sola cadena de transcripción renderizada. Un proveedor cachea sobre la secuencia inicial de tokens de la petición, así que un primer token distinto — un system prompt diferente — invalida todo el prefijo cacheado. Cada compactación pagaba por tanto el costo completo de procesamiento de prompt por todo el historial repetido dos veces: una para la petición de conversación que disparó la presión, y otra para la llamada de resumen, derrotando la caché justo cuando la conversación es más grande.

## Decisión

La directiva de resumen pasa del **frente** de la petición (un `system` prompt nuevo) al **final** de la conversación (el mensaje `user` final). La llamada auxiliar ahora reproduce verbatim el prefijo de la última petición enrutada y añade una instrucción al final, así que es una genuina extensión-de-prefijo de la petición caliente y el proveedor reutiliza los tokens cacheados.

### `SummarizationInput` lleva el prefijo repetido, no una cadena renderizada

`summarize()` (y el interno `summarizeWithLlm`) toman un `SummarizationInput` — `{ system?, tools?, messages }` — en vez de una cadena de transcripción plana. `region.ts` lo construye desde `session.requestHeader()` (el `system` y `tools` duraderos) más la región sombreada mapeada a través de `session.deriveEventMessage`, lo que produce objetos `Message` byte-idénticos a los que `deriveMessages()` plegó en la petición enrutada. `summarizeWithLlm` reenvía `system` y `tools` a `GenerateOptions` y envía `[...input.messages, { role: 'user', content: COMPACTION_INSTRUCTION }]`. Los `tools` van junto aunque el resumidor jamás llame uno: soltarlos acortaría la secuencia de tokens y rompería la alineación con la petición cacheada.

### La instrucción es un mensaje user al final

`COMPACTION_INSTRUCTION` abre «You are now acting as a compaction engine…» y dirige al modelo a condensar *la conversación de ARRIBA*. Conserva los encabezados estructurados del checkpoint previo y añade dos reglas que el system prompt frontal no necesitaba en su nueva posición: no mencionar la petición de resumen, y emitir solo el texto del checkpoint sin llamar a una herramienta. La región sombreada siempre termina en un límite balanceado de emparejamiento de herramientas, así que añadir un mensaje `user` después es un ordenamiento válido de mensajes para los adaptadores OpenAI-compatible y DeepSeek.

### La reutilización de caché es best-effort; la corrección no

La auto-compactación siempre ancla en la cabeza superficial, así que la región sombreada es la cabeza de la petición enrutada y el prefijo repetido coincide exactamente — el caso de acierto garantizado. Un `compactRegion` manual de rango medio sigue repitiendo el prefijo verdadero y permanece correcto, pero renuncia a la reutilización porque su región sombreada no es la cabeza de la petición. Un `summarizationProvider`/`summarizationModel` configurado que difiera de la ruta de la conversación también renuncia a la reutilización; ese es el trade-off explícito del despliegue, no un defecto. La resolución de objetivo (override configurado → último header enrutado → opciones del agente, si no, lanzar) no cambia.

## Alternativas consideradas

- **Conservar el system prompt del resumidor pero reutilizar el resto** — descartado: el slot system es la primera región de tokens sobre la que un proveedor cachea, así que un system prompt de resumidor distinto invalida todo el prefijo independientemente de lo que siga. Solo moviendo la directiva fuera del frente se recupera la caché.
- **Enviar solo la región sombreada sin la cabeza `system`/`tools`** — descartado: una secuencia con cabeza diferente diverge igualmente de la petición cacheada en el primer token, así que no cachea mejor mientras pierde el encuadre que el resumen necesita.
- **Omitir `tools` de la petición de resumen** (el modelo jamás llama una) — descartado: los esquemas de herramientas son parte de la secuencia de tokens cacheada; omitirlos desalinea cada token siguiente y derrote la reutilización.
- **Una sub-sesión de resumen dedicada que emite `assistant/chunk` para el replay de instantáneas** — descartado: el evento duradero `compaction/summary` registra la posición y salida completa de la llamada local exitosa, mientras que su marcador de llamada explícito evita que el replay trate la plantilla o salida remota como un stream local.

## Consecuencias

- **`dsh-compaction-basic`** posee `SummarizationInput`; la firma protegida `summarize(input, agent, signal?)` cambió (aceptable pre-release), y `region.ts` ganó `buildSummarizationInput` plegando `deriveEventMessage` sobre los seqs sombreados detrás del prefijo de header.
- **Superficie de render muerta eliminada.** La vieja ruta de aplanamiento (`renderTranscript` / `renderContentBlocks` y su spec en `dsh-compaction`) no tenía consumidor restante y se borró con su export.
- **La experiencia de modelo del README** de `dsh-compaction-basic` ahora documenta la petición auxiliar como el prefijo repetido más un mensaje de instrucción de compactación al final, y su efecto en caché KV como reutilización del prefijo caliente de la conversación.
- **La salida del checkpoint encuadrado no cambia**, así que el `user/message` aterrizado y cada instantánea de petición de conversación quedan sin afectar; solo cambió la forma de la petición auxiliar.

## Testing

- **Unit:** `compaction-basic.spec.ts` aserta que la llamada auxiliar reenvía `system`/`tools`/mensajes iniciales y añade la instrucción de compactación como mensaje final, y que `compactRegion` repite el prefijo del último header enrutado. Las aserciones de contenido existentes leen la entrada del resumidor a través de los mensajes repetidos y no de una cadena de transcripción.
- **Loop:** `compact-loop-repro.spec.ts` clasifica la petición de resumen por la instrucción de compactación en su mensaje user final, y los tests de recuperación de overflow siguen fijando los conteos de peticiones conversación-vs-resumen a través del bucle real.
- **Snapshot:** el replay sin clave reconstruye un stream exitoso canónico desde un `compaction/summary` marcado; la [nota del seam de compactación](../feature/2026-06-18-compaction-capability-seam.es.md) posee el contrato del marcador duradero.
