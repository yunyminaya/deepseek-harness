---
name: dsh-translate-docs
description: Ejecuta manualmente el flujo ampliado bilingüe de documentos de DeepSeek Harness, incluidos briefings generados, traducción de prosa delegada, traducción de documentos completos y verificación acotada del emparejamiento.
disable-model-invocation: true
user-invocable: true
---

# Traducir documentación de DeepSeek Harness

[English](SKILL.md) | Español

## Límite de invocación

Ejecuta este flujo ampliado solo cuando el usuario invoque explícitamente `dsh-translate-docs` por nombre.
Nunca lo selecciones ni lo cargues para trabajo ordinario de documentación, desde otro skill o por una necesidad de traducción inferida; la traducción rutinaria sigue la regla de una sola pasada en [docs/AGENTS.md](../../../docs/AGENTS.es.md).

## Qué es este skill

**Este skill es guía, no una translation memory.**
Es el mapa de flujo para mantener coherentes y naturales los pares `foo.md ↔ foo.zh.md` en ambos idiomas.
Ambos idiomas tienen la misma autoridad: un cambio se redacta en cualquiera de ellos y ese lado es la fuente para esa actualización.
Tú eres el traductor: las reglas siguientes dicen qué debe cumplirse, no cómo redactar una frase concreta; el juicio de redacción es tuyo, la terminología no.

## Triaje por tipo de cambio — esto decide todo lo demás

- **Update** (el par existe, un lado fue editado): sigue [la ruta de actualización](#the-update-path-briefing-driven).
Se basa en briefing y es deliberadamente barata: no se relee el corpus de guía, no hay arqueología de git y la edición en la contraparte es la mínima posible.
Nunca retraduzcas un documento completo para aplicar una actualización: una actualización mínima conserva la redacción revisada de todo lo que no cambió; una retraducción tira esa revisión.
- **New pair** (aún no existe contraparte): sigue [la ruta de documento completo](#the-whole-document-path-new-pairs).
- **Deleted or renamed doc**: elimina o renombra junto con él la contraparte y el `.i18n.yaml`; de lo contrario el gate informa un par incompleto.

Las Agent Notes congeladas bajo `.agents/notes/archived/` no son trabajo de traducción.
Sus tripletes completos están sellados por el archive verifier; nunca actualices, re-registres ni repares ninguno de sus lados tras el archivado.

## La ruta de actualización (guiada por briefing)

La ruta guiada por briefing alcanza calidad de guidance corpus a una fracción del coste; la [Agent Note briefed-updates](../../notes/implemented/process/2026-07-26-briefed-minimal-translation-updates.es.md) posee la evidencia de referencia.

1. **Genera el briefing**: `pnpm run gen-translation-brief <any file of the pair>` (sin argumentos hace briefing de todo par fuera de sincronía).
El briefing mapea el cambio con la granularidad alineada más estrecha que sea segura — unidades Markdown cambiadas (párrafo, fila de tabla, ítem de lista, encabezado), luego secciones completas por encabezado y luego documento completo — y contiene el diff del lado redactado desde el último estado confirmado como consistente, la última fuente confirmada, la fuente actual y el texto actual de la contraparte para cada unidad cambiada (con números de línea), las filas de terminología que toca el cambio, notas sobre el movimiento de primeras apariciones y un resumen de las reglas vinculantes de actualización.
2. **¿Diff solo mecánico? Aplícalo con `--apply`.**
Cuando cada cambio cae dentro de bloques de código que el par comparte byte a byte, el briefing lo indica; `pnpm run gen-translation-brief --apply <pair>` inserta los fences editados en la contraparte y valida la estructura del resultado antes de escribir — sin subagente, sin edición manual.
3. **¿Diff de prosa? Delégalo a un subagente pasándole el briefing** (o el comando para generarlo).
El briefing es todo el conjunto de trabajo del traductor — el subagente no relee el corpus de guía (el resumen de reglas, las filas de terminología y el contexto triple de cada unidad cambiada ya van inline) y no vuelve a derivar el diff.
Solo escala a las fuentes de verdad de la ruta de documento completo cuando el briefing deje una decisión concreta realmente sin respuesta — un término no listado sin precedente en el texto circundante, o un briefing de documento completo (`BOTH sides changed`, o cuando ni las unidades ni las secciones alineen), que siempre implica reconciliar a mano bajo [translation-rules.md](../../../docs/i18n/translation-rules.md).
4. **La edición más pequeña que cubra el diff.**
Conserva la redacción revisada de todo lo que el diff no toca y luego verifica los hunks cambiados cláusula por cláusula contra la fuente: nada añadido, nada omitido, terminología según las filas inline y code spans verbatim.
5. **Registra y verifica, con alcance acotado**: `pnpm run verify-translation-pairing --write <pair>` y luego `pnpm run verify-translation-pairing <pair>`.
`--write` nombra exactamente los pares que confirmaste — se niega a ejecutarse sin argumentos para que un re-registro masivo siempre sea un `--all` explícito.
La comprobación de todo el corpus ya se ejecuta en `doc-sync`/CI; no la ejecutes por cada update.

## La ruta de documento completo (pares nuevos)

Cuando las traducciones deban escribirse desde cero, el agente orquestador no traduce: lanza un subagente para que haga el trabajo de traducción.
El traductor lee primero las fuentes de verdad siguientes y luego traduce el archivo completo al otro idioma — sección por sección en documentos largos, manteniendo la estructura de cada sección bloqueada a la fuente mientras avanzas en vez de corregir la estructura al final.

### Fuentes de verdad (léelas, no las vuelvas a resumir)

- **[docs/i18n/README.md](../../../docs/i18n/README.es.md)** — el contrato de emparejamiento: el triplete (`foo.md`, `foo.zh.md`, `foo.i18n.yaml`), los blob hashes de ambos lados en el registro de consistencia, las líneas del conmutador de idioma, el alcance y las exclusiones.
- **[docs/i18n/translation-rules.md](../../../docs/i18n/translation-rules.es.md)** — cómo traducir: fidelidad, preservación de estructura, disciplina terminológica y tipografía (niveles MUST/SHOULD).
- **[docs/i18n/terminology.md](../../../docs/i18n/terminology.es.md)** — la tabla de terminología, vinculante en ambos sentidos.
Cárgala ANTES de traducir, no cuando un término parezca incierto; los términos que no notas son los que se desvían.
- **[docs/i18n/translation-prompt.md](../../../docs/i18n/translation-prompt.es.md)** — la plantilla calibrada consumida por máquina del pipeline automatizado.
Los agentes que usan este skill no la renderizan; la tabla de terminología es el único archivo del repositorio que el renderizador automatizado inyecta, mientras este skill y `translation-rules.md` siguen siendo vinculantes para traducciones redactadas por agentes.
- **[dsh-prose-standard](../dsh-prose-standard/SKILL.es.md)** — cobertura obligatoria de prosa y juicio editorial.
Aplícalo a ambos lados sin añadir ni quitar proposiciones de la fuente.

### Traducir

- **Pass 1 — write, don't transpose.**
Lee una unidad semántica y luego restátala como autor técnico nativo en el registro de la [muestra de estilo](../../../docs/i18n/style-samples.md) más cercana.
Conserva el marco requerido sin forzar correspondencia oración por oración.
- **Pass 2 — verify against the source, clause by clause.**
La fidelidad se comprueba aquí, no se escribe aquí: confirma que no se añadió ni se omitió nada, que cada término siga la tabla y que cada code span haya sobrevivido verbatim.
Corrige reescribiendo la frase de forma nativa, no parcheando palabras dentro de ella.
- **Lee la contraparte completa por sí sola.**
Después de comparar con la fuente, lee el archivo traducido sin la fuente al lado y reescribe la redacción cuya torpeza solo se vuelve visible en aislamiento.
- Escribe al archivo solo el texto final, nunca borradores ni notas.
- Cada término de [terminology.md](../../../docs/i18n/terminology.es.md) se renderiza exactamente como se especifica.
Para un destino chino, usa las columnas Chinese y first-occurrence; un término no listado necesita un precedente citable de OSS/vendor chino o permanece en inglés bajo 「待定术语」.
Para un destino inglés, usa la columna English y un término técnico inglés consolidado; conserva un término fuente ambiguo con una glosa corta y anótalo como pendiente.
Nunca inventes una traducción inline.
- Los bloques de código son byte-idénticos en ambos lados del par, incluidos los comentarios.
Los enlaces de documentos relativos al repositorio conservan el mismo destino semántico y el sufijo exacto de query/fragment: los destinos del corpus bilingüe activo usan `.md` en el lado inglés y `.zh.md` en el lado chino, una contraparte faltante dentro del alcance es un error, los destinos fuera del corpus conservan su ruta original y el switcher sigue siendo la excepción cross-locale.
- El gate de emparejamiento comprueba profundidades de encabezados, fenced blocks, recuentos de filas y columnas de tablas, tipos de listas, arranques de listas ordenadas, recuentos de ítems, locale del enlace y destinos semánticos.
En el Pass 2, verifica manualmente el orden de listas y tablas, la numeración no canónica de listas, el código inline, el énfasis, el significado, la terminología y el tono.

## Encuentra el trabajo

- `pnpm run verify-translation-pairing --list` imprime cada documento dentro del alcance como missing / out-of-sync / ok.
Las filas missing y out-of-sync son violaciones del contrato; la comprobación normal las rechaza.
- `pnpm run gen-translation-brief` sin argumentos imprime el briefing de cada par fuera de sincronía.
- En un PR que edite documentos emparejados, la lista de trabajo es el propio diff: cada lado cambiado de un par necesita su contraparte actualizada y el par re-registrado en el mismo PR, y el gate se pondrá en rojo si lo olvidas.

## Termina el par

1. Switcher: `[English](foo.md) | 中文` inmediatamente tras el H1 del archivo chino, `English | [中文](foo.zh.md)` tras el H1 del archivo inglés — añade ambos si es un par nuevo, salvo cuando una fuente inglesa propiedad de un generador deba seguir byte-idéntica a la salida del generador y omita su switcher mientras la contraparte china sí enlaza de vuelta.
2. Registra la consistencia: `pnpm run verify-translation-pairing --write <pair>` recomputa y registra los blob hashes completos de ambos lados en `foo.i18n.yaml`.
El diff YAML en tu PR es la afirmación revisable “confirmé que estos dos dicen lo mismo” — ejecútalo solo después de confirmarlo de verdad.
3. No hace falta una entrada de manifest para un documento ordinario: toda fuente dentro del alcance requiere un par.
Cambia [scripts/translation-pairing.manifest.json](../../../scripts/translation-pairing.manifest.json) solo cuando la política propietaria documente una exclusión genuina generada, instructiva o bilingüe por construcción.
4. Antes del PR: los pares tocados están en verde bajo la comprobación acotada; `pnpm run doc-sync` (que incluye la comprobación de emparejamiento de todo el corpus más `verify-md-wrap`/`verify-md-links`) se ejecuta una vez a nivel de PR según [dsh-pre-push-checks](../dsh-pre-push-checks/SKILL.es.md), no dentro de cada tarea de traducción.
5. Mantén el PR revisable: indica qué pares son nuevos frente a cuáles fueron mínimamente actualizados y lista de forma visible los 「待定术语」.

## Cómo responder a la review de traducción

Sigue la [guía de reporte de code review](../dsh-code-review/SKILL.es.md#reporting-findings): evalúa cada comentario por sus méritos y, en los comentarios terminológicos, recuerda que la tabla de terminología es el contrato — aplica la decisión de redacción del reviewer a [terminology.md](../../../docs/i18n/terminology.es.md), no solo a un archivo.
