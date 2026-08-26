# Agent Note: Citar artefactos confirmados, nunca ordinales de sesión de diseño

Status: implemented

[English](2026-08-09-committed-artifact-citations.md) | [中文](2026-08-09-committed-artifact-citations.zh.md) | Español

## Problema

Las sesiones grandes de diseño y revisión dejan taquigrafía de trabajo — ordinales de decisión, códigos de elementos de auditoría, números de sección de planes, ordinales de tareas y de la pila, fallos de revisores — que se lee con naturalidad mientras el transcript de la sesión está abierto y no resuelve nada cuando se cierra. Una auditoría de todo el repositorio encontró el patrón concentrado en `packages/client`: citas desnudas `(decision 12/16/19/20/21)` de las cuales solo la decisión 21 tenía dueño confirmado, códigos `(audit C2/S1/S3/S7)` sin ningún documento de auditoría en ningún sitio, referencias `design §4.7` / `web2 §0` / `plan §1.4` a borradores sin confirmar, etiquetas de fase de plan (`T2/T5/T9`, `P-I`, `W5`), posiciones de pila («a later PR in this stack») en JSDoc duradero, y vocabulario de «ruling» / «design ledger». Las mismas familias aparecían en pruebas, comentarios CSS, plantillas de generadores, comentarios de CI y Agent Notes (vantage de «this PR/branch/review round», atribuciones de coreografía de revisión, afirmaciones obsoletas de «deferred to a later PR» cuyos objetivos ya se habían publicado). El [estándar de documentación](../../../../docs/AGENTS.es.md) ya prohibía la mitad de historial de cambios (previously/now, referencias a PR y commits) pero no establecía una regla equivalente para las citas, así que los ordinales irresolubles seguían llegando.

## Decisión

La prosa duradera — comentarios, JSDoc, docs, notas, comentarios y títulos de pruebas — cita solo artefactos confirmados, resolubles dentro del repositorio sin arqueología de grep:

- Nombra la Agent Note propietaria (su ruta al menos una vez por archivo, un nombre buscable en línea), la ruta de la página de docs o un número de incidencia de GitHub. Las posiciones de PR, commit, rama y pila siguen prohibidas en docs y código según el estándar de documentación; las incidencias son duraderas y citables, y las Agent Notes y los post-mortems pueden citar PRs fusionados e incidencias como evidencia, según el enrutamiento de historias de cambio del [estándar de documentación](../../../../docs/AGENTS.es.md).
- Un ordinal de sesión de diseño cuya decisión tiene dueño confirmado se sustituye por el nombre de la decisión — el ordinal que se registró como «decision 21» ahora es «la decisión de referencia en texto plano», propiedad de la [nota de la input-machine web](../architecture/2026-07-25-web-input-machine-and-slash-pipeline.es.md); el ordinal en sí no resuelve nada dentro del repositorio y se eliminó en todas partes. Un ordinal sin dueño se borra y su cláusula factual se reformula para sostenerse sola.
- Las regresiones corregidas se fijan como contrafactuales en presente («without X, Y happens»; «a naive X would…»), nunca como historial del repositorio («used to Y»).
- Las notas implementadas declaran la realidad publicada: una afirmación de «deferred to a later PR» cuyo objetivo ya se publicó nombra la nota publicada en su lugar.
- Los fixtures registrados, las instantáneas y las notas archivadas están exentos: la salida de modelo registrada y el historial sellado conservan su voz original. Dentro de las secciones de historial de cambios de una nota, un nombre de etapa histórica («the first cut shipped X») es seguro para el estado actual; los sellos deícticos («this cut») siguen prohibidos en todas partes.

Una purga en todo el repositorio aplicó estas reglas en las superficies de prosa, incluidas las plantillas propiedad de generadores (`scripts/gen-doc-graphs.ts`, `scripts/gen-tool-catalog.ts`, el aviso de página del generador typert) con regeneración, el JSDoc fuente type-equiv con nuevos pegado de páginas, y los homólogos bilingües con nuevos registros de par. El [skill dsh-trim-cot-leakage](../../../skills/dsh-trim-cot-leakage/SKILL.es.md) operacionaliza estas reglas: la taxonomía de auditoría, las baterías de recuerdo confirmadas y los ejemplos few-shot para decidir qué conservar o borrar.

## Alternativas consideradas

- **Confirmar los design ledgers y los documentos de auditoría para que los ordinales resuelvan.** Rechazado: los transcripts de sesión son artefactos de trabajo, no referencias mantenidas; confirmarlos habría creado un corpus de decisiones paralelo y sin puertas junto a las Agent Notes, y su numeración interna seguiría derivando.
- **Una puerta mecánica para el vocabulario prohibido.** Diferido: el vocabulario es lenguaje natural sin límites, y las baterías de recuerdo de la auditoría necesitan criterio para separar la fuga de la prosa legítima («wait» como sustantivo, «actually» contrastivo, estados viejos/nuevos en runtime). Una puerta estrecha de alta precisión (por ejemplo `\(decision \d`, `\(audit [A-Z]\d`, `\bcut \d`, `this cut`, un `\bT\d\b` desnudo, `P-I`, `used to `, un `\bv1\b` desnudo y `§\d` — esta última excluyendo las citas cuya numeración de sección tiene dueño confirmado, como el propio §N de web-styling.md) es el candidato si el patrón reaparece; la revisión de la propia purga encontró residuales exactamente en los casos que esas búsquedas no detectaban, así que lideran la lista de candidatos.
- **Borrar la justificación que citaba artefactos muertos.** Rechazado: las cláusulas factuales se conservaron o reformularon; solo se eliminaron las citas, la coreografía de revisión y los transcripts de derivación, según la regla de proposición completa del estándar de prosa.

## Verificación

Las baterías de grep de la auditoría (inglés y chino, comentarios y prosa, `--hidden` para `.agents/`) no devuelven citas de ordinales de diseño fuera de los fixtures registrados, las notas archivadas, los propios archivos del skill trim y la evidencia citada de esta nota; `verify-type-equiv`, las comprobaciones de frescura `gen-*` y `verify-translation-pairing` fijan las superficies regeneradas y re-registradas. Hueco de cobertura: ninguna puerta rechaza una cita de ordinal nueva — la revisión es dueña de la regla.

## Consecuencias

- Las citas de un comentario se resuelven por ruta o nombre; los lectores nunca reconstruyen una sesión cerrada para seguir una.
- Las sesiones de diseño deben aterrizar sus decisiones en Agent Notes antes de que la prosa duradera pueda citarlas; la taquigrafía de ordinales se queda dentro de la sesión.
- Las citas se alargan (una ruta de nota en lugar de «(decision 21)») a cambio de una resolución sin grep.
