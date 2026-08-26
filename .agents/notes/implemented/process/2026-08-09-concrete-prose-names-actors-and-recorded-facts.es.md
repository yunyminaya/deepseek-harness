# Agent Note: La prosa concreta nombra actores y hechos registrados

Status: implemented

[English](2026-08-09-concrete-prose-names-actors-and-recorded-facts.md) | [中文](2026-08-09-concrete-prose-names-actors-and-recorded-facts.zh.md) | Español

## Problema

La prosa del repositorio usaba etiquetas de categoría abstractas donde los lectores necesitaban hechos concretos distintos. La misma etiqueta podía significar secuencias de eventos anteriores citadas por un reemplazo, el provider y el modelo que produjeron un mensaje, el llamador que suministró el contexto, el archivo que suministró una fila de configuración o el trabajo de CI que compiló un binario. Los lectores tenían que inspeccionar el código antes de poder distinguir qué hecho prometía la oración.

Sustituir una etiqueta amplia por otra habría conservado esa ambigüedad. Renombrar tipos, campos y miembros de protocolo durante una limpieza editorial habría cambiado en cambio contratos que el problema de redacción no exigía cambiar.

## Decisión

La prosa mantenida nombra el actor, la acción, la fuente, el evento, el campo, el archivo o el proceso exactos que necesita el contrato local. Declara qué se registró y quién o qué lo registró. Los redactores aplican una comprobación de lenguaje hablado y sustituyen las palabras que no usarían mientras explican el mismo punto a un colega.

La regla se aplica a Markdown, READMEs, Agent Notes activas, JSDoc y comentarios, prompts, diagnósticos y cadenas visibles para el usuario. Una auditoría juzga cada oración por separado; no sustituye un término en todo el repositorio por un sinónimo preferido. La oración editada conserva actor, acción, condiciones, orden, modalidad, excepciones, titularidad, comportamiento de fallo y consecuencias.

Los identificadores de código exactos, las APIs públicas, los campos duraderos, los miembros de protocolo, los nombres de tipo, los encabezados con referencias externas y los nombres de archivo permanecen sin cambios salvo que un cambio de nombre de contrato coordinado sea independientemente necesario. La prosa circundante explica sus campos o comportamiento directamente. Los documentos y catálogos generados se actualizan desde su fuente propietaria.

Antes de usar `contract`, `boundary` o `shape`, los redactores comprueban si la oración significa una regla, operación, estructura de datos, conjunto de campos, punto de validación, punto de temporización, API, tipo o condición de fallo más específicos. `Contract` sigue siendo correcto para precondiciones, postcondiciones, invariantes, promesas de compatibilidad y otras obligaciones en las que confían llamadores, callees, implementadores, providers, productores o consumidores. `Boundary` sigue siendo correcto para una división literal de seguridad, confianza, cable, proceso, serialización, transacción o ciclo de vida. `Shape` sigue siendo correcto cuando la propia forma estructural es el sujeto y ningún término más estrecho como campos, schema, tipo, variante de unión, disposición de archivo o forma de exportación declara el hecho. Los nombres de código y API que contienen estas palabras permanecen sin cambios salvo que se requiera un cambio de nombre coordinado aparte.

Esta decisión complementa la decisión de [niveles y presupuestos de documentación](2026-07-04-doc-tiers-and-budgets.es.md), que sigue siendo dueña de la ubicación, la forma del documento y los presupuestos de palabras.

## Alternativas consideradas

**Prohibir una lista fija de palabras.** Rechazado porque una palabra puede ser un identificador exacto o el término más claro en otro contrato. Por ejemplo, los invariantes llamador/callee son contratos reales, y los límites de proceso o de cable identifican divisiones reales. La revisión oración a oración detecta la ambigüedad sin rechazar nombres válidos.

**Sustituir cada etiqueta abstracta por «source», «origin» o «metadata».** Rechazado porque otra etiqueta amplia sigue dejando que los lectores infieran si la oración significa un archivo, un llamador, una secuencia de eventos, un par provider/modelo, un commit o un trabajo de compilación.

**Renombrar cada identificador coincidente junto con la prosa.** Rechazado porque la claridad editorial no justifica migraciones no relacionadas de API, protocolo, formato duradero, tipo o archivo. Esos cambios exigen su propia auditoría de consumidores y su propia decisión.

## Consecuencias

La documentación y los diagnósticos pueden usar unas palabras más, pero cada afirmación dice a los lectores qué valor o proceso importa sin exigir inspección de la fuente. Las auditorías de prosa en todo el repositorio exigen clasificación semántica y no pueden usar sustitución a ciegas. Los homólogos bilingües conservan el mismo hecho concreto, y las copias generadas se refrescan solo después de que cambie su fuente propietaria.
