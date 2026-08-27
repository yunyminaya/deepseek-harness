---
name: dsh-trim-cot-leakage
description: Úsalo al auditar o corregir prosa que suena como un reasoning transcript filtrado — citas muertas de sesiones de diseño como (decision N), códigos de ítems de auditoría o §N de borradores no commiteados; narración de cambios como “used to”, “no longer”, “this cut”; perspectiva de stack o review (“a later PR in this stack”, “rejected in review”); justificaciones dirigidas al reviewer; narración de flujo de control; o residuos de planificación dubitativa en comentarios, JSDoc, docs o Agent Notes.
---

# Recortar fugas de Chain-of-Thought

[English](SKILL.md) | Español

Una fuga de chain-of-thought es prosa cuya perspectiva es la sesión de autoría y no el repositorio: cita artefactos que solo esa sesión podía ver, narra el cambio en vez del estado o discute con un reviewer que ya se fue.
La corrección nunca consiste solo en borrar cuando un pasaje contiene cláusulas factuales: vuelve a declarar cada una para que se sostenga en HEAD y luego elimina la transcripción que la rodea; un pasaje que no lleva ninguna (un código de auditoría, narración de flujo de control) se elimina por completo.
**CONTEXTO OBLIGATORIO:** [dsh-prose-standard](../dsh-prose-standard/SKILL.es.md) posee la regla de proposición completa que este skill aplica; la [nota de citas de artefactos commiteados](../../notes/implemented/process/2026-08-09-committed-artifact-citations.es.md) posee la justificación de la regla de citas.
Es guía, no script.

## La única prueba

Para cada pasaje sospechoso pregunta: **¿podría un lector en HEAD, sin acceso a ninguna transcripción de sesión, hilo de PR o borrador no commiteado, resolver cada referencia y verificar cada afirmación?**
Si no, vuelve a declarar los hechos supervivientes desde la perspectiva del repositorio y elimina el resto.
Si sí, no es una fuga, por muy histórico que suene — pero la resolubilidad solo supera el listón de este skill: en superficies de estado actual (READMEs, docs, JSDoc) una historia de cambio resoluble sigue siendo narración de cambio, y la clase 3 la manda a su hogar sancionado.

## Taxonomía

1. **Citas muertas de sesiones de diseño** — `(decision 7)`, `(audit C2)`, `design §4.7`, `plan §1.4`, etiquetas de fase (`T4`, `W3`, `P-I`), “the design ledger”, “(B ruling)”.
Si la decisión tiene un propietario commiteado, cítala por nombre y ruta; si no, elimina la cita y vuelve a declarar su cláusula factual para que se sostenga sola.
2. **Perspectiva de stack y PR** — “a later PR in this stack”, “this PR adds”, “the previous commit”.
Declara el mecanismo enviado o el punto de extensión; el trabajo aplazado va a un marcador `TODO` o a una referencia de issue.
3. **Narración de cambio y sellos de versión** — “used to”, “no longer”, “the old X” y sellos indexicales (“v1”, “this cut”, “today”, “now” en contraste con un estado pasado).
Declara el comportamiento presente; una regresión corregida se convierte en un contrafactual en tiempo presente (“without X, Y happens”), nunca en arqueología del repo (“used to Y”).
4. **Coreografía de review** — “Rejected in review:”, confirmaciones del reviewer, ordinales de draft (“v5 of this note”), atribuciones de ronda.
Conserva la decisión y la justificación supervivientes como hecho llano; elimina quién lo dijo.
5. **Justificación dirigida al reviewer** — “the cast is safe — it simply…”, “this is correct because…”.
Un comentario que defiende su propia corrección se dirige a un reviewer, no a un maintainer.
Declara el invariant que hace seguro al código, o elimina el comentario si el código ya lo muestra.
6. **Reformulación y transcripciones de derivación** — narración de flujo de control (“first we X, then we Y”), walkthroughs de tests, pruebas de ramas obvias.
Elimínalas; conserva solo un contrato o invariant no obvio.
7. **Coberturas dubitativas y residuos de planificación** — “probably fine for now”, “should be enough”, aplazamientos sin marcador.
Promuévelos a `TODO`/`FIXME` o vuelve a declararlos como el límite real; elimina la duda.
8. **Deslices de idioma de autoría** — fragmentos no traducidos del idioma de trabajo (端, 设计稿, separadores `---- 私有 ----`) en una prosa por lo demás inglesa, o lo inverso en una contraparte zh.
Traduce o elimina.

## Qué no es una fuga

Las pasadas sin ayuda fallan en ambos sentidos al borrar referencias duraderas y conservar las muertas.
Aplica estas reglas de conservación tal como están escritas; [examples](references/examples.es.md) calibra cada una:

- **Referencias a issues** — `#1470`, `TODO(name):`, “issue #N owns the follow-up” resuelven en HEAD; consérvalas en cualquier superficie, incluidos READMEs.
No las reubiques a Agent Notes.
- **Citas a PRs fusionados e issues dentro de Agent Notes y postmortems** — evidencia sancionada según el enrutado de historias de cambio del [estándar de documentación](../../../docs/AGENTS.es.md).
- **Justificaciones de supresión** — `oxlint-disable … -- reason`, razones de coverage-ignore y explicaciones de empty-catch son prosa obligatoria; corrige una razón falsa, nunca la elimines.
- **Anclajes de regresión en presente contrafactual** — “without X, Y happens”, “a naive X would…”.
- **Límites medidos** — “(measured: 512 nests ≈ 0.15s)” calibrando una constante; la palabra de procedencia “measured” soporta carga.
- **Estados old/new de runtime** — “the old connection drains before the new one accepts” es ciclo de vida en runtime, no historia de cambios.
- **Nombres de etapas históricas dentro de las secciones de historia de cambio de una nota** — “the first cut shipped X” es seguro respecto al estado actual allí; los sellos indexicales (“this cut”) siguen prohibidos en todas partes.
- **Referencias externas que resuelven fuera del repo por diseño** — secciones de estándares (RFC 9110 §10.1.5), nombres de frames de Figma; la prohibición de `§` cubre borradores internos no commiteados, no estándares externos ni docs commiteados que posean su propia numeración por `§`.
- **Voz y formas de género del proyecto** — “we” como voz del proyecto; la sección Alternatives considered de una nota.

## Flujo

1. Alcance y exclusiones según [dsh-prose-standard](../dsh-prose-standard/SKILL.es.md): exige un alcance explícito; nunca toques `vendor/`, `.agents/notes/archived/` ni fixtures y snapshots grabados — la salida de modelo registrada y la historia sellada conservan su voz original.
2. Audita primero en solo lectura: ejecuta las [recall batteries](references/recall-batteries.md) (con `--hidden` para que se busque `.agents/`) y luego juzga cada coincidencia semánticamente.
Las batteries son sondas, no la definición — cada ronda de review de la purga original encontró casos que las batteries no detectaron, así que también debes leer la prosa más densa dentro del alcance (JSDoc de módulo, READMEs, Agent Notes) sin depender de un patrón previo.
3. Corrige con owner-first según la superficie: catálogos generados → corrige la fuente JSDoc o la plantilla del generador y luego regenera; fences de equivalencia de tipo → corrige el JSDoc fuente y luego vuelve a pegar ambas páginas bilingües (`verify-type-equiv` las fija); pares bilingües → actualiza la contraparte y vuelve a registrar según [dsh-translate-docs](../dsh-translate-docs/SKILL.es.md); cadenas visibles para el modelo → el redactado es comportamiento, así que márcalo para un cambio respaldado por snapshot en vez de reescribirlo en silencio.
4. Antes de borrar nada, enumera las proposiciones del pasaje (prose-standard) y comprueba las [trampas de sobrecorrección](references/examples.es.md#overcorrection-traps): recortes que convierten una obligación en una aprobación, promueven un hipotético a funcionalidad enviada, borran un hecho verdadero o eliminan procedencia.
5. Verifica: vuelve a ejecutar las batteries esperando solo conservaciones sancionadas, el propio directorio de este skill y la evidencia citada del note propietario; confirma que cada cita restante resuelva en HEAD; ejecuta los gates de las superficies tocadas (`doc-sync` para docs, `verify-type-equiv`, `verify-translation-pairing`).
