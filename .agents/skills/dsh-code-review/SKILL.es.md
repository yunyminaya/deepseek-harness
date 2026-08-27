---
name: dsh-code-review
description: Úsalo al revisar un pull request en el repo deepseek-harness — orienta al reviewer hacia los estándares de este codebase (convenciones de AGENTS.md, patrones defensivos, ADRs, quality gates) y hacia las comprobaciones específicas de review que el código por sí solo no puede mostrar.
---

# Revisar un PR de DeepSeek Harness

[English](SKILL.md) | Español

**Este skill es guía, no una checklist completa.**
Verifica y trae la base viva del PR y el head exacto, luego ejecuta `pnpm --silent run change-scope --base <verified-base-ref> --head <verified-head-ref>` antes de leer el diff y suficiente código circundante para entender el diseño.
El informe identifica rutas y capas sucias, pero no sustituye la review semántica.
Vuelve a establecer la base y reejecútalo tras un retarget o un merge.
Prioriza corrección, ciclo de vida, seguridad y comportamiento obligatorio roto por encima del estilo; una review breve con un bloqueo fundamentado es mejor que una lista de nimiedades.

## Fuentes de verdad

- [AGENTS.md](../../../AGENTS.md) y [packages/AGENTS.md](../../../packages/AGENTS.md): reglas permanentes del repositorio y de autoría de paquetes.
- [docs/defensive-patterns.md](../../../docs/defensive-patterns.es.md): clases de bugs de subprocess, callback, async-state y disposal.
- [docs/AGENTS.md](../../../docs/AGENTS.es.md): ubicación de documentación y disciplina de prosa.
- [dsh-prose-standard](../dsh-prose-standard/SKILL.es.md): cobertura obligatoria y criterio editorial para comentarios, docs, prompts y cadenas visibles.
- [docs/testing.md](../../../docs/testing.es.md) y la [Agent Note de quality gates](../../notes/implemented/process/2026-06-11-quality-gates.es.md): tiers y gates de prueba requeridos.
- [Agent Notes](../../notes/README.es.md): justificación de diseño.
Trata un desacuerdo con una Agent Note como una discusión de diseño, no como un veto automático.
- Para cambios bilingües, lee [translation-rules.md](../../../docs/i18n/translation-rules.es.md) y [terminology.md](../../../docs/i18n/terminology.es.md); el skill ampliado de traducción queda fuera de la review automática y solo se ejecuta por invocación explícita del usuario.

## Requisitos bloqueantes

1. **La prosa nueva recibe review semántica.**
Usa [dsh-prose-standard](../dsh-prose-standard/SKILL.es.md) para revisar críticamente todo pasaje Markdown añadido o cambiado, JSDoc, comentario, prompt, descripción, diagnóstico y cadena visible.
Verifica cobertura requerida, exactitud, ubicación y calidad editorial contra el código o comportamiento propietario; las comprobaciones automatizadas no establecen esas propiedades.
2. **La documentación coincide con el código.**
Config, defaults, errores, campos wire, eventos y comportamiento público actualizan el README del paquete y el JSDoc en el mismo diff.
Los comentarios declaran contratos no obvios; marca para borrar la narración de implementación, walkthroughs de tests, historia de review y justificación duplicada, o enlázalos a su único hogar.
3. **La documentación de tipos del core coincide.**
Los cambios al vocabulario spine o seam actualizan la página apropiada de [subsystems](../../../docs/subsystems/README.es.md) y cualquier entrada `type-equiv`.
Los tipos internos no necesitan una entrada de catálogo.
4. **Los registros se limpian.**
Verifica que cada contribución nueva a un registry pase las pruebas de disposal requeridas por [packages/AGENTS.md](../../../packages/AGENTS.md).
5. **Los invariant companions son semánticos.**
Para cada `./invariant` tocado, exige una relación de event-stream propietario o de datos mutables en el punto donde ese paquete pueda observarla; presencia de servicios o métodos, metadata o effects de plugins y ejemplos puros fijos pertenecen a pruebas de tipo, carga o unidad.
Acepta un installer vacío cuando su razón específica de paquete establezca que no existe una relación plausible en runtime; no exijas una comprobación inventada solo para eliminar el vacío ([regla del repositorio](../../../AGENTS.md#conventions); [reglas de invariants de paquetes](../../../packages/AGENTS.md)).
6. **Existe evidencia obligatoria.**
Verifica que el autor ejecutó las [comprobaciones locales relevantes](../../../AGENTS.md#run-relevant-checks-locally) para el diff y que la CI cubre la matriz exhaustiva; revisa las lagunas semánticas que ninguno de los dos puede detectar.

## Comprobaciones manuales

- **Contratos de intención e interfaz:** traza ambos lados de cada interfaz cambiada.
Confirma que la implementación coincida con el PR y con cualquier Agent Note, incluidos errores, cancelación, propiedad y disposal.
- **Ciclo de vida y concurrencia:** para setup asíncrono, callbacks, procesos o teardown, aplica [defensive-patterns.md](../../../docs/defensive-patterns.es.md).
Comprueba carreras antes de la publicación, cancelación durante awaits, reporte independiente de errores, contención de callbacks, propiedad antes del reentry, limpieza completa del detach y disposal en quiescencia.
- **Ajuste entre capability y consumer:** traza cada consumer actual y luego marca comportamiento específico del consumer que se filtra a la interfaz bajo [las reglas de paquetes](../../../packages/AGENTS.md).
Marca también lo inverso: un método público nuevo en un servicio genérico (registry, session, agent) cuyo único caller sea un consumer interno es una expansión innecesaria de API — exige en su lugar un cierre de capability privado entregado a ese consumer en la construcción.
- **Alcance, propiedad y necesidad:** mapea cada abstracción, state machine, opción, copia defensiva y ruta de compatibilidad a su contrato actual, consumer de producción y plugin o servicio propietario.
Cuestiona funcionalidades no relacionadas y generalidad especulativa, y luego contrasta el PR con [las reglas raíz](../../../AGENTS.md#conventions).
- **Configuración y elecciones públicas:** pregunta qué evidencia de consumer actual o precedente sustenta cada default, conjunto de operaciones públicas, formato o concepto externo importado.
Exige una elección explícita o un aplazamiento cuando falte esa evidencia.
- **Perspectiva del modelo:** inspecciona los prompts exactos, tool schemas, resultados y diagnósticos que recibe el modelo en los modos afectados.
Marca conceptos fuera de la tarea del modelo y luego verifica texto estable verbatim y comportamiento dinámico mediante snapshots o cobertura end-to-end.
- **Enforcement:** sigue cada ruta de denegación hasta la operación que la ejecuta; ejercita callers directos y alternativos que puedan eludir schemas, prompts, facades, wrappers o el orden de listeners.
- **Estado prestado y derivado:** determina si cada valor retenido es prestado o poseído bajo el contrato del paquete y luego traza notificaciones y cada caché, prompt, eco de UI, replay y vista de consulta hasta el punto documentado de éxito y la fuente autoritativa.
- **Los límites cubren la operación final:** localiza al propietario del resultado emitido o retenido completo, incluidos wrappers y metadata.
Prueba límites mínimos y exactos, chunks únicos sobredimensionados y texto multibyte para límites en bytes.
- **Ruta real de entrada:** los tests ejercitan el loader, bin, worker, ACP bridge o subprocess enviado cuando sea relevante.
Un plugin montado a mano no detecta exports inválidos del Loader; un function plugin debe exportar con nombre su namespace y no tener default export.
- **Fuerza de tests:** las assertions fallan ante la regresión prevista y verifican estado externo, logs, eventos o disposal en vez de reformular la implementación o confiar en el reporte de un agent.
La cobertura es necesaria, pero no prueba que el escenario sea correcto.
- **Ciclo de vida de invariants y controles negativos:** verifica que las observaciones candidatas se rechacen antes de la publicación cuando sea posible, que las comprobaciones respaldadas por sesión reconstruyan historia duradera tras carga tardía o HMR, y que un caso deliberadamente inválido falle a través del runner real para la regla prevista.
- **Las Agent Notes implementadas coinciden con la realidad enviada:** cuando un PR implementa una Agent Note propuesta, muévela y reescríbela como estado enviado en tiempo presente dentro del mismo diff, y luego verifica rutas, nombres y mecanismos contra la implementación.
- **Cambios de transcript:** los cambios visibles para editor o modelo actualizan snapshots o explican por qué no aplica ninguno.
Revisa los diffs de salida esperada como cambios de comportamiento, no como ruido de formato.
- **Cambios bilingües:** compara significado y terminología en ambos lados; un pairing hash en verde no prueba calidad de traducción.

## Informar hallazgos

Declara el defecto, la ubicación, el impacto y la evidencia.
Coloca un defecto localizado inline sobre el rango de diff relevante más estrecho; usa un comentario a nivel de PR para arquitectura transversal, alcance o síntesis general de la review.
Separa bloqueos de sugerencias y omite problemas ya forzados por un gate en verde.
Usa el hilo de review existente de GitHub para responder.
Al recibir review, verifica cada afirmación y corrígela o rebátela por motivos técnicos sin acuerdo performativo.
