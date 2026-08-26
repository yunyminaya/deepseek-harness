# Agent Note: Contrato del prompt de traducción calibrado v4

Status: implemented

[English](2026-07-23-translation-prompt-v4-contract.md) | [中文](2026-07-23-translation-prompt-v4-contract.zh.md) | Español

## Problema

La generación automatizada de contrapartes necesita un prompt estable que reproduzca el registro estilístico y las correcciones establecidas por las traducciones revisadas por humanos. Inyectar un documento de instrucciones de propósito general cambia esa entrada calibrada del modelo cada vez que cambia la guía para humanos o agentes, mientras que una respuesta sin envoltorio no puede transportar por separado un borrador, su autorevisión y el documento corregido. Además, las etiquetas de sección planas tipo XML colisionan con el Markdown válido que documenta esas mismas etiquetas.

## Decisión

El [prompt de traducción](../../../../docs/i18n/translation-prompt.es.md) consignado en el repositorio es el activo calibrado del pipeline. Su renderizador inyecta únicamente el idioma de origen, el idioma de destino y la [tabla de terminología](../../../../docs/i18n/terminology.es.md) vigente, y rechaza sintaxis de marcador de posición desconocida, ausente o malformada antes de ensamblar una solicitud. El ensamblador de solicitudes retiene el nombre base de la fuente fuera del prompt visible para el modelo y coloca cada par revisado de documento completo en un turno de ejemplo usuario/asistente de texto plano antes del documento fuente real. La plantilla puede portar reglas de calibración específicas del modelo, pero esas reglas permanecen subordinadas a los contratos de emparejamiento, terminología, estructura y énfasis del repositorio.

La calibración v7 retiene ese protocolo v4 y hace explícita la prioridad de las instrucciones: el significado de la fuente y la estructura protegida, luego la tabla de terminología, luego la voz de los pares ejemplares de documento completo, y después la guía general y los ejemplos incrustados. Dirige al modelo a redactar como un autor técnico nativo y después comparar cláusula por cláusula, preservando actores, condiciones, negación, modalidad, condiciones de ciclo de vida, dirección, canales de resultado, titularidad y cantidades. La guía de estilo no puede inventar un actor ni variar una forma de la tabla de terminología, un concepto definido o un verbo de contrato meramente por variedad. La terminología sin resolver permanece sin cambios en la traducción y solo se reporta como pendiente de revisión.

La respuesta tiene tres secciones ordenadas de nivel superior: `translation`, `review` y `final`. El consumidor de la respuesta deriva el nombre base de destino a partir del contexto de la fuente retenido, preserva el frontmatter YAML inicial opcional e inserta o corrige mecánicamente el conmutador de idioma tras el primer H1 en `final`. El analizador exige cada sección exactamente una vez, rechaza el contenido fuera del envoltorio y tolera una cerca Markdown `xml` externa porque los modelos a veces repiten la cerca de ejemplo del prompt.

## Encuadre de la respuesta

Las líneas delimitadoras de sección están reservadas por el formato de transmisión. Cuando una línea del cuerpo Markdown consiste en una etiqueta delimitadora, posiblemente precedida de barras invertidas, el serializador y el modelo añaden una barra invertida inicial; el analizador elimina exactamente una. Este escape que preserva el conteo hace el recorrido de ida y vuelta tanto con un delimitador literal como con un delimitador ya escapado, sin alterar las menciones de etiquetas en línea.

El contrato ejecutable vive en [el renderizador, el ensamblador de solicitudes, el analizador y el consumidor de respuesta](../../../../scripts/translation-prompt.ts). Las pruebas unitarias cubren ambas direcciones, el orden de la solicitud, la validación de marcadores de posición, la validación de la ruta de destino, el orden y la cardinalidad estrictos de las secciones, las respuestas cercadas, las menciones de etiquetas en línea, las líneas delimitadoras dentro de cuerpos Markdown y la corrección del conmutador de nuevos pares que preserva el frontmatter. Una instantánea de subproceso sin clave fija el prompt ensamblado y cinco turnos de ejemplo revisados junto con una respuesta grabada que porta frontmatter, consumida a través de la corrección de ruta de destino.

## Alternativas consideradas

**Inyectar `translation-rules.md` en cada solicitud.** Ese documento rige tanto a humanos y agentes como al pipeline automatizado. Inyectarlo acopla cada aclaración editorial al comportamiento del modelo y desplaza las restricciones del prompt calibradas manualmente; el pipeline, en cambio, inyecta la tabla de terminología vinculante y verifica su propio activo directamente.

**Usar un documento XML estricto con CDATA.** CDATA provee un encuadre XML general pero añade un protocolo anidado, un escape `]]>` adicional y el comportamiento de un analizador XML que el contrato de tres secciones no necesita de otra forma. Reservar y escapar seis líneas delimitadoras conserva las secciones calibradas de la respuesta a la vez que preserva Markdown arbitrario.

**Devolver solo la traducción final.** Un cuerpo único es más sencillo de analizar pero descarta la pasada explícita de corrección que se usa para capturar defectos de tono, estructura, terminología y puntuación antes de la publicación.

**Reemplazar íntegramente el activo calibrado v4 por un prompt experimental posterior.** Los experimentos posteriores aclararon reglas generales útiles pero no superaron la evaluación estricta de documento completo como reemplazos totales. El activo de producción adopta solo las mejoras que preservan los ejemplos establecidos y el protocolo ejecutable.

**Exigir un registro estricto de cambios del borrador al final.** Exigir que cada edición final aparezca en un registro de revisión de texto libre añade carga de salida sin demostrar que la revisión capture la deriva semántica local. La revisión registra las correcciones reales, mientras que la traducción final sigue sujeta a comprobaciones deterministas de estructura y a revisión humana.

## Consecuencias

La redacción del prompt es comportamiento ejecutable y recibe revisión de código, un verificador de prompt de traducción y una instantánea ejecutable de solicitud/respuesta. Pruebas enfocadas fijan los ejemplos incrustados y salvaguardas v7 seleccionadas. El directorio de instantáneas `translation-prompt-v4` nombra la serie estable del protocolo renderizador/analizador, no la revisión de calibración vigente. El activo calibrado y las reglas generales de traducción pueden evolucionar para sus distintas audiencias, pero la revisión debe rechazar contradicciones con los contratos del repositorio. El escape de línea es visible solo cuando la documentación fuente contiene una etiqueta envolvente en su propia línea, y las pruebas del analizador fijan su comportamiento sin pérdida.
