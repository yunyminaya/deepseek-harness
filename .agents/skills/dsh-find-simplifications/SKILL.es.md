---
name: dsh-find-simplifications
description: Úsalo al trabajar en el repo deepseek-harness para encontrar candidatos de simplificación no obvios, escribir Agent Notes propuestas o notas inline TODO/FIXME/XXX, auditar o fusionar Agent Notes sustituidas, o incorporar ideas de simplificación valiosas desde otro PR; especialmente en superficies muertas, duplicadas, especulativas, sobrediseñadas, añadidas-y-luego-eliminadas o hechas a mano donde ya existe una dependencia adecuada.
---

# Encontrar simplificaciones en DeepSeek Harness

[English](SKILL.md) | Español

Este skill ayuda a convertir una petición amplia de «encuentra cosas para simplificar» en Agent Notes respaldadas por evidencia que eliminen o colapsen superficie existente del harness.
Es guía, no checklist: sigue el código, mantén el juicio activo y prefiere unos pocos candidatos bien probados a un montón de conjeturas débiles.

## Empieza con el contexto del repo

- Lee `AGENTS.md`, especialmente la postura pre-release y las convenciones (incluidas las doctrinas tests-are-not-golden-truth y Agent Notes-are-not-golden-truth), además de [docs/defensive-patterns.md](../../../docs/defensive-patterns.es.md) y [docs/testing.md](../../../docs/testing.es.md).
- Revisa por encima [docs/architecture.md](../../../docs/architecture.es.md) antes de juzgar cualquier cosa bajo `packages/`; las simplificaciones que chocan con el mapa de servicios o la taxonomía de eventos necesitan evidencia adicional.
- Usa el árbol de Agent Notes y sus [reglas](../../notes/README.es.md) para entender la arquitectura intencional.
Los ejemplos implementados más relevantes son [drop mutable session summary](../../notes/implemented/simplification/2026-06-19-drop-mutable-session-summary.es.md), [shared persistence write coordinator](../../notes/implemented/architecture/2026-06-18-shared-persistence-write-coordinator.es.md), [capability seams](../../notes/implemented/architecture/2026-06-13-capability-seams.es.md) y las Agent Notes de twin adapter / dual persistence backend.
- Trata los adaptadores LLM duales y los backends de persistencia duales como intencionales por defecto.
No propongas eliminar ninguno de los twins/backends como «low effort» salvo que el usuario anule explícitamente esa restricción.
Eliminar un método u hook sin uso dentro de un seam protegido puede seguir siendo válido si no colapsa el diseño protegido.

## Qué cuenta como candidato fuerte

Un candidato fuerte elimina, pliega o rebaja algo real y tiene evidencia clara de que el diseño actual cuesta más de lo que aporta:

- Un método público, evento, config knob, notificación de registro, helper, paquete, evento duradero o artefacto de test no tiene consumer de producción.
- Tests o docs son los únicos consumers, y el comportamiento que fijan no soporta carga.
- Dos representaciones reflejan el mismo hecho, especialmente entre eventos duraderos de sesión y eventos transitorios `agent/*`.
- Un seam tiene métodos que toda implementación debe soportar pero que ningún consumer usa.
- Un paquete separado existe solo para código de test/demo/support y añade sobrecarga de publicación o dependencias.
- Una funcionalidad implementa generalidad especulativa del producto: multi-session/session-load, rosters de background jobs, invalidación en vivo del registro, steering a mitad de turno, renderizado de UI propiedad de la herramienta y diseños similares sin product owner.
- Un invariant, ruta de rollback, conjunto de expected outputs o test de caso especial existe solo para proteger una API sin uso.
- Código hecho a mano reimplementa algo que ya ofrece un paquete npm bien mantenido o un builtin de Node disponible en el engine floor, y el cambio eliminaría la implementación más sus tests dedicados ([política de dependencias](../../notes/implemented/process/2026-07-26-dependencies-over-hand-rolling.es.md)).
- El comportamiento simplificado puede diferir un poco, pero el nuevo comportamiento sigue siendo razonable y más fácil de explicar.

Los candidatos débiles normalmente no bastan para una Agent Note: borrar una errata, ejecutar `knip` una vez, eliminar un backend/adaptador documentado intencionalmente o señalar «esto parece complejo» sin prueba en call sites.

## Explora con amplitud

Usa subagentes en paralelo cuando el usuario pida amplitud o muchos candidatos.
Da a cada agent un dominio y exige evidencia, no conjeturas.
Dominios útiles:

- Agent loop y session log: límites de turno/paso, steering, abort/cancel, eventos duraderos, replay, load/resume.
- ACP automation y APIs de UI humana: liquidación de prompt y teardown en el lado del protocolo; renderizado de transcripción y estado de interacción en el lado de la UI.
- LLM/tools/system prompt: APIs stream/generate, assemblers, registros, defaults de tool schema, presentation hooks.
- Bash y ejecución de herramientas: división foreground/background, propiedad de jobs, archivos spill de salida, métodos del ejecutor.
- Packages/examples/scripts/tests: divisiones de paquetes, inventarios estáticos, expected outputs redundantes, paquetes de soporte.

Si los subagentes no están disponibles, simula tú mismo la misma amplitud.
No permitas que el primer buen candidato detenga la exploración.

Empieza por los deltas más grandes de código de producción.
Una auditoría amplia de simplificación que se detiene tras símbolos sin uso obvios puede perder los archivos donde la maquinaria de ciclo de vida o defensiva duplicada carga la mayor parte del coste.

## Audita límites de confianza y ciclo de vida

Para cada copia defensiva, freeze, validador y captura de callback, nombra de dónde vino el valor y quién lo posee después.
Las llamadas tipadas de servicio/plugin en el mismo proceso normalmente toman prestados valores readonly; parsers, cargadores de config, queues, JSON de modelo/herramienta, archivos duraderos, workers, procesos y decodificadores wire poseen o validan sus datos.
Los tests construidos alrededor de getters hostiles, objetos tipados falsos, reemplazo de callbacks o mutación tras un handoff en el mismo proceso son evidencia de un contrato posiblemente especulativo, no justificación automática para conservarlo.

Para código asíncrono complejo, dibuja el grafo de propiedad y asigna cada sentinel, readiness promise, ruta de cancelación, disposer y flag de estado a un propietario o transición distintos.
Cuando varios mecanismos reflejen el mismo hecho de vivacidad o liquidación, propone una transacción o controlador de ciclo de vida único en su lugar.
Conserva maquinaria separada donde proteja publicación y rollback síncronos, contención de callbacks, arbitraje del primer resultado terminal, propiedad de worker/process o dispose-to-quiescence.

## Código hecho a mano frente a una dependencia

Introducir una dependencia es un movimiento válido de simplificación, no una excepción política: la [política de dependencias](../../notes/implemented/process/2026-07-26-dependencies-over-hand-rolling.es.md) fija el listón.
Al explorar, pregunta en parsers de protocolo, framers, bucles retry/backoff, comparadores glob, motores de diff e infraestructura similar: ¿un paquete npm bien mantenido o un builtin de Node en el engine floor ya hace esto?

Demuestra un candidato de intercambio por dependencia como cualquier otro, además de:

- Lee la implementación hecha a mano y nombra la superficie exacta que cubre el paquete; la semántica residual que el paquete no cubra cuenta en contra del cambio y debe permanecer en la Agent Note.
- Comprueba honestamente la salud del paquete (mantenimiento, adopción, huella transitiva) y prefiere builtins cuando el engine floor ya los tenga.
- Comprueba primero el árbol de Agent Notes: schemastery, Cordis vendorizado, los twin adapters y otros seams registrados están resueltos; un cambio que colapse uno de ellos debe superar la justificación registrada, no solo citar la política.
- Sopesa la eliminación neta: implementación más tests dedicados más docs, menos el pegamento que permanezca.
Un wrapper que solo reubica la misma complejidad no es una ganancia.

## Demuestra o rechaza cada candidato

Para cada símbolo o comportamiento, clasifica los consumers antes de escribir:

- Corpus de producción: `packages/*/src`, `examples/*/src`, `examples/**/*.yml`, scripts de runtime y rutas del loader/config.
- Corpus no productivo: tests, README/docs, Agent Notes, snapshots, expected outputs generados y comentarios.
- Corpus ambiguo: examples y scripts que podrían ser rutas de humo del producto.
Inspecciona su uso antes de clasificar.

Usa `rg` primero.
Buenas búsquedas incluyen el símbolo exacto, el nombre del evento, el nombre del paquete, la clave de config, el nombre del método con tanto `.name(` como `name(`, y cualquier string wire.
Luego lee los call sites.
`knip` puede ayudar, pero no sustituye entender interfaces públicas, nombres de eventos dinámicos, tests, docs y rutas del loader Cordis.

Rechaza o rebaja un candidato cuando:

- Existe un caller de producción y la simplificación sería una decisión de funcionalidad en lugar de una limpieza.
- La API está justificada explícitamente por una Agent Note implementada o por un defensive pattern ganado a pulso, y la nueva evidencia no supera esa razón.
- La eliminación forzaría churn no relacionado sin reducir realmente la API pública ni el comportamiento obligatorio.
- La idea es correcta pero pequeña.
Añade en su lugar un TODO/FIXME/XXX dirigido, usando la semántica de urgencia de [docs/development.md](../../../docs/development.es.md).

## Fusiona Agent Notes reemplazadas

Audita el árbol de Agent Notes cuando el usuario pida reducirlo o fusionarlo, o cuando la simplificación implementada vuelva obsoleta una nota propietaria.
No conviertas toda exploración de simplificación de código en una auditoría a escala de repositorio.

Usa [`dsh-archive-agent-notes`](../dsh-archive-agent-notes/SKILL.es.md) para el juicio de retención y la mecánica de archivo.
Las notas implementadas de poco valor futuro se mueven como tripletes congelados a `archived/{kind}`; las propuestas nunca se archivan; las rechazadas que ya no impiden un error tentador se eliminan.
No edites una nota archivada mientras simplificas prosa o código actuales.

Sigue la regla de eliminación de las [reglas de Agent Notes](../../notes/README.es.md#when-to-write-one); no la dupliques ni la debilites aquí.
Para cada cadena candidata:

1. Identifica al propietario actual desde el código enviado, la configuración, los catálogos generados, la documentación de paquetes, Agent Notes más nuevas y enlaces entrantes; las fechas y títulos son ayudas de descubrimiento, no prueba.
2. Clasifica la nota antigua como sustituida total o parcialmente.
Cualquier comportamiento superviviente, contrato actual, formato duradero, obligación de compatibilidad o alternativa rechazada aún vigente la hace parcial.
La justificación transferible al propietario actual no hace parcial por sí sola la sustitución.
3. Para sustitución total, mueve al propietario actual toda justificación única, alternativa, consecuencia, evidencia de verificación enviada y laguna de cobertura con nombre.
Un inventario que solo describa mecánica de implementación eliminada no es uno de esos hechos de decisión.
4. Repara cada enlace entrante y luego elimina juntos la nota inglesa, la contraparte china y el registro de consistencia.
5. Busca por nombres exactos de archivo, símbolos, claves de config, nombres de eventos y strings wire después de editar.
Mantén las sustituciones parciales enlazadas entre sí y actualizadas.

Una funcionalidad añadida y luego eliminada es un caso común de sustitución total.
Deja que la nota de eliminación posea la historia solo cuando la funcionalidad esté ausente del código de producción, la configuración, schemas, formatos duraderos o wire, migración y comportamiento de compatibilidad; ninguna documentación actual la presente como disponible; y ningún test la ejercite como comportamiento soportado.
La justificación de la eliminación y los tests que hacen cumplir la ausencia pueden permanecer.
Conserva por qué la funcionalidad existió originalmente, por qué esa motivación ya no justificaba mantenerla, las alternativas a la eliminación total, la capacidad que se pierde, las condiciones para reintroducirla y la evidencia de que la eliminación está completa.
Los tests antiguos y la mecánica de implementación que verificaban solo el comportamiento eliminado no son evidencia actual de verificación.

Rechaza la consolidación cuando la eliminación sea solo un transporte, default, implementación o presentación de una funcionalidad; cuando sobrevivan datos persistidos o manejo de compatibilidad; o cuando la nota de eliminación aún no lleve suficiente justificación para evitar una reintroducción accidental.
Una decisión de diseño negativa y actual puede necesitar legítimamente su propia nota aunque la implementación eliminada ya no exista.

## Escribe la Agent Note

Crea un archivo por propuesta duradera bajo `.agents/notes/<lifecycle>/<class>/yyyy-mm-dd-topic.md`, siguiendo las reglas de ciclo de vida y clasificación de `.agents/notes/README.es.md`.
Mantén los párrafos en una sola línea física y usa enlaces Markdown relativos.

Prefiere esta estructura, ajustándola cuando la idea lo necesite:

- `# Agent Note: <título orientado a la acción>`
- `Status: proposed`
- `## Problem`: nombra la API actual, cita los archivos relevantes y declara la evidencia de consumers.
Separa callers de producción de tests/docs.
- `## Proposal`: di exactamente qué eliminar, plegar, rebajar o reubicar.
Incluye limpieza de tests, docs, READMEs, JSDoc, taxonomía de eventos, snapshots y archivos generados cuando sea relevante.
- `## Why not keep it?` o `## What we give up`: haz legible el contraargumento más fuerte.
- `## Acceptance criteria`: estado final observable y gates.
- `## Risks`: cambios en API pública, cambios de comportamiento, necesidades futuras del producto y por qué el intercambio sigue siendo razonable.

Sé lo bastante concreto como para que un PR de implementación pueda seguir el rastro.
Evita Agent Notes vagas tipo «simplifica este paquete».
Cuando una propuesta se solape con una Agent Note existente, consolida los detalles útiles en la nota existente que posee el tema en lugar de crear un duplicado.

## Notas TODO inline

Usa TODO/FIXME/XXX inline solo para limpiezas pequeñas y locales claramente útiles pero que no sean decisiones de diseño duraderas.
Mantenlas cortas y accionables:

- Nombra el olor con una etiqueta estable, por ejemplo `TODO(double-default)` o `XXX(unused-default)`.
- Explica por qué es seguro revisitarlo y qué acción lo simplificaría.
- No añadas TODOs para quejas especulativas ni para comportamientos que requieran una decisión al nivel de Agent Note.

## Al incorporar otro PR o rama

Haz diff de la rama hermana contra `origin/master`, no contra la rama del PR actual, para ver su contribución independiente.
Para cada elemento:

- Porta Agent Notes o TODOs no solapados que cumplan el listón de calidad.
- Consolida el material solapado en la Agent Note existente que posee el tema.
- No portes propuestas duplicadas o de menor confianza solo para conservar el recuento.
- Actualiza el cuerpo del PR para que reviewers vean el recuento y el alcance reales de candidatos.
- Cierra el PR duplicado solo cuando el usuario te lo haya pedido o cuando esté claro que tú posees esa tarea de housekeeping.

## Validación e higiene de PR

Para trabajo solo de Agent Notes en docs, ejecuta al menos `pnpm run doc-sync`, `pnpm run lint` y `git diff --check`.
Para comentarios de código o cambios de skills, ejecuta además el validador relevante cuando exista.
Selecciona cualquier otra evidencia a partir del diff saliente; el hook pre-push aporta typecheck únicamente.

Al abrir o actualizar un PR, resume:

- Cuántas Agent Notes y notas inline se añadieron, consolidaron, conservaron como sustituciones parciales o eliminaron.
- Las áreas principales exploradas.
- Qué se excluyó intencionalmente.
- Qué comprobaciones pasaron.

Para cada grupo de consolidación, nombra los propietarios antiguo y actual, declara la evidencia de sustitución total y explica por qué eliminar es seguro.
Si una exploración de añadido-y-luego-eliminado no encuentra ninguna nota que califique, informa ese resultado y los casos parciales representativos que se conservaron.

Usa un draft PR mientras la exploración siga ampliándose; márcalo como listo solo cuando el conjunto de candidatos, las respuestas de review y la validación estén cerrados.
