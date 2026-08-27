---
name: dsh-archive-agent-notes
description: Úsalo al añadir, auditar, podar, archivar, restaurar o revisar Agent Notes en deepseek-harness; comprueba cada nota nueva en busca de registros activos reemplazados, clasifica las notas implementadas por su valor para decisiones futuras, elimina notas rechazadas que ya no impiden una falacia tentadora y aplica el triplete congelado archived/{kind} y las reglas del manifest.
---

# Archivar Agent Notes de DeepSeek Harness

[English](SKILL.md) | Español

Reduce el corpus activo de decisiones sin borrar historia que todavía pueda guiar el trabajo.
Juzga cada nota semánticamente; el recuento de palabras y la antigüedad son ayudas de descubrimiento, nunca criterios de archivo.

## Lee los contratos

Lee [las reglas de Agent Notes](../../notes/README.es.md), [las instrucciones de archivo](../../notes/archived/AGENTS.md) y las instrucciones vigentes del ciclo de vida aplicable antes de clasificar.
Usa el código actual, la configuración, la documentación de paquetes, los catálogos generados, Agent Notes más nuevas y los enlaces entrantes para establecer si una justificación todavía posee o restringe algo.

## Comprueba la sustitución al añadir una nota

Cada Agent Note nueva activa una auditoría acotada de notas activas que cubren la misma decisión, mecanismo o alternativa rechazada.
Clasifica cada sustitución total o parcial mientras escribes la nota nueva: archiva en el mismo PR los tripletes implementados que califiquen, conserva y enlaza las sustituciones parciales o la justificación que siga siendo útil por sí misma, rechaza las propuestas obsoletas y elimina las notas rechazadas que ya no evitan un error plausible.
Aplica la regla de consolidación de Agent Notes cuando el nuevo propietario absorba toda proposición única; no difieras una coincidencia conocida a una auditoría posterior del corpus.

## Clasifica por valor futuro

Aplica estos resultados específicos del ciclo de vida:

- **Implementada — mantener activa:** conserva una nota cuando su justificación, alternativas, garantías negativas, semántica duradera/de wire, límite de propiedad, regla de seguridad o condición de reintroducción probablemente guíen un cambio futuro.
La longitud no importa.
- **Implementada — archivar:** archiva una nota cuando la decisión enviada esté completa y su cuerpo sea poco probable que guíe trabajo futuro, como chrome de UI de una sola vez, un adaptador estrecho, un bug cerrado menor, detalle de implementación reemplazado o historia de proceso cuyo comportamiento actual ya resulta obvio en otro sitio.
- **Propuesta — nunca archivar:** mantén activa una propuesta viva; si ya no merece perseguirse, recházala con una razón honesta y cumple el formato del ciclo de vida rechazado.
- **Rechazada — conservar solo como guardarraíl:** conserva un rechazo solo cuando la propuesta perdedora siga siendo un error tentador y significativo y la nota explique por qué pierde.
- **Rechazada — eliminar:** elimina el triplete completo cuando la idea rechazada sea obsoleta, sustituida, ya no plausible o poco probable que evite reabrir el debate.
Repara o elimina los enlaces entrantes.

No archives para alcanzar una cuota.
Inspecciona cada nota dentro del alcance, clasifica grupos análogos bajo un mismo principio, usa tu mejor juicio en los casos cercanos y registra para el handoff las decisiones realmente limítrofes.

## Ejemplos calibrados

Estos ejemplos fijan el listón; los recuentos de palabras muestran que el tamaño no es la prueba.

Archiva notas implementadas como:

- collapsed sidebar control rail — 533 palabras: comportamiento de UI menor y cerrado;
- Commander argument adapter — 1.498 palabras: detalle sustancial de implementación con poco apalancamiento futuro de diseño;
- documentation graph atlas — 920 palabras: maquinaria de documentación completada cuyos generadores actuales son la autoridad.

Conserva activas notas implementadas como:

- event-sourced sessions — 248 palabras: autoridad fundacional y límite de durabilidad;
- single Harness-home resolver — 596 palabras: regla de propiedad entre productos;
- project session directories — 628 palabras: política de almacenamiento duradero e identidad;
- parallel pre-push gates — 400 palabras: caso limítrofe, pero aún guía la programación de gates y el ajuste de recursos;
- dropped image content block — 334 palabras: conservar hasta que llegue el soporte multimodal, porque fija la condición coordinada de reintroducción.

Para notas rechazadas:

- conservar folding the compaction package split — 426 palabras: la tentación de fusionar los paquetes sigue siendo significativa;
- eliminar streaming workflow progress through tool calls — 972 palabras: su premisa ACP/UI está obsoleta;
- eliminar dropping ACP terminal metadata — 362 palabras: la decisión posterior de ACP solo para automatización resolvió la cuestión.

## Archivar un triplete implementado

1. Mueve el triplete completo `foo.md`, `foo.zh.md` y `foo.i18n.yaml` desde `implemented/<kind>/` a `archived/<kind>/`; `implemented` está ausente deliberadamente de la ruta de archivo.
2. No edites el cuerpo.
Inserta solo `Archived: YYYY-MM-DD` inmediatamente debajo de `Status: implemented` en ambos archivos de idioma, usando la fecha de archivado y el mismo valor en ambos lados.
3. Vuelve a registrar mecánicamente los hashes del sidecar para las dos ediciones solo de metadatos.
No traduzcas, reformatees, actualices hechos ni repares enlaces dentro de la nota.
4. Busca enlaces entrantes desde prosa activa.
Redirígelos a la autoridad actual, reapúntalos a la ruta archivada solo cuando la instantánea histórica se cite intencionalmente o elimínalos.
Nunca verifiques ni repares enlaces salientes de la nota archivada.
5. Ejecuta `pnpm run verify-archived-agent-notes --write`.
Su modo append-only primero demuestra que cada sello existente sigue coincidiendo y luego añade solo los hashes del triplete nuevo.
Ejecuta después el verificador normal.

Tras sellar el triplete, nunca lo edites, muevas, traduzcas, reformatees ni elimines.
Las notas archivadas siguen siendo destinos válidos para enlaces entrantes, pero son instantáneas históricas, no autoridad sobre el comportamiento actual.

## Validar e informar

Ejecuta la prueba enfocada del verificador de archivo, `pnpm run verify-archived-agent-notes`, `pnpm run doc-sync`, `pnpm run lint` y `git diff --check`; selecciona cualquier evidencia adicional con [dsh-pre-push-checks](../dsh-pre-push-checks/SKILL.es.md).

Informa las notas implementadas activas conservadas, las implementadas archivadas, las rechazadas conservadas/eliminadas, las propuestas rechazadas si las hay, y cada caso realmente limítrofe con su recuento de palabras y el resultado elegido.
No afirmes que los enlaces salientes archivados son válidos: el verificador de archivo intencionalmente nunca los comprueba.
