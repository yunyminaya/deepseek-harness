# Ejemplos destilados de fugas

[English](examples.md) | Español

Destilados de la purga de todo el repo de 2026-08 y de sus rondas de review.
Úsalos para identificar el principio rector, no como plantillas de texto.
Este archivo cita deliberadamente redacción filtrada como material de calibración — las [recall batteries](recall-batteries.md) excluyen el directorio de este skill y su redactado no concede licencia para usarlo en otro lugar.

## Citas muertas

### Ordinal de decisión con propietario commiteado

**Filtrado:** “Slash input resolves against the visible catalog (decision 21).”

**Corregido:** “Slash input resolves against the visible catalog — the plain-text-reference decision, owned by [the web input-machine note](../../../notes/implemented/architecture/2026-07-25-web-input-machine-and-slash-pipeline.es.md).”

El ordinal no resuelve a nada en HEAD; el nombre de la decisión y la ruta de su nota propietaria sí.
Nombra la ruta de la nota propietaria al menos una vez por archivo — como enlace cuando la superficie lo soporte — y las menciones posteriores pueden usar solo el nombre buscable.

### Ordinal de decisión sin propietario

**Filtrado:** “The registry rejects duplicate names (decision 7: names are flat, no namespacing).”

**Corregido:** “The registry rejects duplicate names; names are flat, with no namespacing.”

Ningún artefacto commiteado posee “decision 7”, así que la cita se elimina — pero su cláusula factual (nombres planos) se vuelve a declarar para que se sostenga sola, no se borra con ella.

### Códigos de ítems de auditoría

**Filtrado:** “Rendering is pure: same snapshot, same string (audit R3).”

**Corregido:** “Rendering is pure: same snapshot, same string.”

No existe ningún documento de auditoría en el repo; el código es una abreviatura de sesión sin proposiciones.

### Números de sección de borradores no commiteados

**Filtrado:** “Layering follows the design (v2 §3.2): `src/core/` is the pure core.”

**Corregido:** “Layering: `src/core/` is the pure core.”

`§N` de un borrador que nadie commiteó no es resoluble.
En cambio, “escapes per RFC 9110 §10.1.5” se conserva — un estándar externo resuelve fuera del repo por diseño, y un doc commiteado que posea su propia numeración por `§` puede citarse por sección.

### Etiquetas de fase del plan

**Filtrado:** “`src/client/` is the shell (T4); the P-I migration owns the adapters.”

**Corregido:** “`src/client/` is the shell; the adapters live in `src/client/adapters/`."

Las etiquetas de fase indexan un plan que nunca aterrizó.
Sustituye la etiqueta por lo que produjo esa fase.

## Perspectiva de stack y PR

### Posición en un stack dentro de prosa duradera

**Filtrado:** “A future remote backend implements this interface (the sandbox backend is a later PR in this stack).”

**Corregido:** “A remote backend can implement this interface without changing the render layer.”

La prosa duradera no puede ver el stack.
Conserva el contrato del punto de extensión; el hogar del trabajo pendiente es el propio PR, un `TODO` o un issue.

### “This PR” en un README

**Filtrado:** “This PR adds cursor-based pagination to the session list.”

**Corregido:** “The session list paginates by cursor.”

Un README sobrevive a todos los PRs; declara el mecanismo como hecho actual.

## Narración de cambios y sellos de versión

### War story con número de PR

**Filtrado:** “Colors used to come from `--widget-*` tokens, which nothing defined, so it always rendered the fallbacks; the alias tokens fixed that (PR #88).”

**Corregido:** “Colors come from the alias tokens; an undefined token renders the fallbacks.”

Ambos hechos vivos sobreviven — el mecanismo actual y el comportamiento persistente ante fallo — reformulados en presente.
La biografía del bug pertenece al PR y a su Agent Note.

### Narración de una eliminación

**Filtrado:** “The `probe` field is gone with the removal cut; badges ride the generic projection pair now.”

**Corregido:** “Badges use the generic projection pair.”

Quien nunca vio `probe` no aprende nada de su ausencia.
“Now” en contraste con un pasado borrado es un sello de versión.

### Regresión corregida → presente contrafactual

**Filtrado:** “This used to double-encode multibyte labels.”

**Corregido:** “Without the byte-length guard, multibyte labels double-encode.”

El anclaje de regresión sobrevive como un contrafactual en presente que nombra el guard; “used to” lo fija a la arqueología del repo.

### Sellos indexicales de versión

**Filtrado:** “Batch rendering is synchronous this cut; the async path is roadmap work.”

**Corregido:** “Batch rendering is synchronous.” (El aplazamiento vive en `TODO(widget-batch):` en el call site.)

“This cut” / “v1” / “today” caducan en cuanto se fusionan.
Un nombre de etapa histórica dentro de la sección de historia de cambio de una Agent Note (“the first cut shipped X”) es seguro respecto al estado actual; la forma indexical nunca lo es.

## Coreografía de review

### Veredictos de review como prosa

**Filtrado:** “Rejected in review: caching the resolved spec. We keep resolution per-call.”

**Corregido (en Alternatives considered de una Agent Note):** “**Caching the resolved spec.** Rejected: the spec depends on per-call cwd, so a cache keyed by request would serve stale roots.”

El género Alternatives considered es el hogar sancionado; el reviewer y la ronda no forman parte de la justificación.

### Ordinales de draft

**Filtrado:** “As of v5 of this note, the loader also validates manifests.”

**Corregido:** “The loader validates manifests.”

Una nota implementada declara realidad enviada; su propia historia de revisiones vive en git.

## Justificación dirigida al reviewer

### Defender un cast

**Filtrado:** “The cast is safe — the SDK constructed the object, it simply doesn't declare the optionals strictly enough.”

**Corregido:** “The SDK constructs this object with every optional populated; the declared type is looser than the runtime guarantee.”

Declara el invariant que un maintainer no debe romper.
“It simply…” es una voz que responde a una objeción que nadie en HEAD planteó.
Si el invariant es visible en el código, elimina el comentario en su lugar.

### Apelar a la autoridad de la review

**Filtrado:** “This is correct because the reviewer confirmed the wrapping order.”

**Corregido:** (eliminado; el orden de wrapping se declara en el `@returns` de la función.)

Las afirmaciones de corrección citan invariants o tests, nunca personas.

## Reformulación y derivación

### Narración de flujo de control

**Filtrado:** “First we normalize the label, then we truncate it, then we wrap it.”

**Corregido:** (eliminado.)

Las tres líneas bajo el comentario ya dicen lo mismo en código.

### Walkthrough de test

**Filtrado:** “This test creates a session, sends two messages, waits for the second reply, and then asserts the log has four entries.”

**Corregido:** “Two round-trips must produce exactly four log entries — the projection dedupes the shared prefix.”

Conserva solo la justificación no obvia de la assertion; el walkthrough reformula el cuerpo del test.

## Coberturas dubitativas y residuos de planificación

### Aplazamiento sin marcar

**Filtrado:** “Probably fine to render eagerly for now.”

**Corregido:** (eliminado; el aplazamiento ya tiene su marcador `TODO(widget-batch):`.)

Una duda sin propietario es residuo de planificación.
Si no existe marcador, escribe uno (`TODO(name): coalesce per animation frame`) en lugar de conservar la duda.

### Dimensionado vago

**Filtrado:** “A 64 KiB buffer should be enough for most cases.”

**Corregido:** “64 KiB holds the largest observed frame (48 KiB) with headroom; a larger frame fails loudly in `decode`."

Sustituye la duda por el límite real y el comportamiento de fallo cuando se excede.

## Deslices de idioma de autoría

**Filtrado:** “The renderer runs on the client 端; see the 设计稿 for spacing. ---- 私有 ----”

**Corregido:** “The renderer runs on the client side; spacing follows the Figma frame `widget-badges`."

Los fragmentos del idioma de trabajo y los separadores de sesión son residuos de transcripción.
El nombre del frame de Figma se conserva: procedencia externa que resuelve fuera del repo por diseño.

## Conservaciones

### Las referencias a issues son duraderas en cualquier superficie

**Conservar:** “The cap applies to the complete rendered value, wrappers included (issue #1470 owns the follow-up).”

Una pasada sin ayuda eliminó esto razonando que las citas a issues pertenecen a Agent Notes.
Dirección errónea: los issues resuelven en HEAD desde cualquier superficie y “#N owns the follow-up” es el hogar sancionado del trabajo aplazado en un README.
Lo que además sancionan Agent Notes y postmortems es citar *PRs fusionados* como evidencia.

### Las menciones muertas no equivalen a “nombrar al propietario”

**Eliminar:** “Badge renderer over the widget seam (see the widget-rendering RFC).”

Una pasada sin ayuda conservó esto como “nombrar el documento propietario por tema”.
La prueba es la resolubilidad, no la forma: ningún archivo commiteado responde a “the widget-rendering RFC”, así que el puntero está muerto.
Reapúntalo al propietario commiteado si existe; si no, elimínalo.

### Justificaciones de supresión

**Conservar (tras corregir):** `// oxlint-disable-next-line no-non-null-assertion -- the one-element literal guarantees index 0.`

La cláusula de justificación es prosa obligatoria.
Cuando la razón declarada es falsa (la original decía “the loop guard above proves a frame exists” sin haber ningún loop a la vista), corrige la razón; nunca la elimines.

### Límites medidos

**Conservar:** “Depth cap (measured: 512 nests ≈ 0.15s synchronous; 4096 blocks the loop).”

La medición fija la constante frente a reajustes desinformados y “measured” es la procedencia que distingue datos de conjetura.

### Old/new de runtime no es historia de cambios

**Conservar:** “The old connection drains before the new one accepts.”

“Old” y “new” aquí nombran dos objetos vivos de runtime durante el relevo, no estados del repositorio.
La prohibición de narración de cambio trata sobre historia del repo, no sobre vocabulario de ciclo de vida.

## Trampas de sobrecorrección

Cada trampa de abajo se envió en la purga original y se detectó en review.
Enumera las proposiciones del pasaje antes de recortarlo.

### Convertir una obligación en una aprobación

**Original:** “These direct registrations are exceptions pending migration to slots.”

**Sobrecorregido:** “These direct registrations are sanctioned exceptions.”

**Correcto:** “These direct registrations are exceptions pending migration to slots.”

“Pending migration” es una obligación; “sanctioned” bendice el estado actual.
El recorte invirtió la modalidad de la frase mientras la hacía más corta.

### Promover un hipotético a funcionalidad enviada

**Original:** “A future IPC-based shell subclasses the executor and overrides `spawn`."

**Sobrecorregido:** “An IPC-based shell subclasses the executor and overrides `spawn`."

**Correcto:** “A hypothetical IPC-based shell — no such shell exists — would subclass the executor and override `spawn`."

Eliminar solo el marcador de futuro convierte una ilustración de diseño en una afirmación de que la clase se envía.
Marca el hipotético explícitamente en lugar de solo quitar el futuro.

### Borrar un hecho verdadero junto con la transcripción que lo rodea

**Original:** “The gate notice narrates the check order; the notice text is also what `verify-doc-typecheck` compiles against.”

**Sobrecorregido:** “…” (frase completa eliminada como narración.)

**Correcto:** “The notice text is what `verify-doc-typecheck` compiles against.”

La mitad de la frase era narración; la otra mitad era un acoplamiento que soporta carga.
Elimina cláusulas, no frases completas, cuando varias proposiciones comparten una línea.

### Eliminar la procedencia conservando el número

**Original:** “The 4 MiB ceiling is measured: the largest generated `py-types` module is 3.1 MiB.”

**Sobrecorregido:** “The ceiling is 4 MiB; the largest generated `py-types` module is 3.1 MiB.”

**Correcto:** conserva “measured”.

Sin “measured”, 3.1 MiB se lee como una definición y no como una observación, y nadie vuelve a medir antes de subir el techo.
