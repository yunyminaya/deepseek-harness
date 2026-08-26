# AGENTS.md — El estándar de documentación

[English](AGENTS.md) | Español

Este archivo define la estructura de los documentos, los niveles de Markdown, las reglas de redacción y los techos de `verify-doc-budgets`. Usa [dsh-doc-standards](../.agents/skills/dsh-doc-standards/SKILL.es.md) para la ubicación y la validación, y [dsh-prose-standard](../.agents/skills/dsh-prose-standard/SKILL.es.md) para la cobertura requerida y el criterio editorial; el [doc-tiers Agent Note](../.agents/notes/implemented/process/2026-07-04-doc-tiers-and-budgets.es.md) es dueño de la justificación.

## Estructura de los documentos

Estas reglas se aplican a la documentación orientada a humanos; los [Agent Notes](../.agents/notes/README.es.md) quedan fuera de su alcance. Un [post-mortem](postmortem/README.es.md) es una referencia delimitada a un incidente; la cronología registra evidencia, no una secuencia didáctica. El tema de un documento y su posición en el árbol fijan su alcance: describe su propio tema con el detalle apropiado y dirige a los hijos solo por propósito, responsabilidad y comportamiento de alto nivel; enlaza al descendiente propietario para el detalle de nivel inferior. El tipo de documento no amplía ese alcance. Una referencia solo puede ser exhaustiva sobre su propio tema. Los mecanismos de test, los fixtures y los harnesses pertenecen al nivel propietario más bajo; los documentos superiores enlazan allí.

Clasifica cada documento dentro del alcance como tutorial o referencia. Los tutoriales siguen un camino ordenado hacia un resultado e introducen solo lo que cada paso necesita. Las referencias definen un ámbito de consulta y el comportamiento actual sin secuencia didáctica. Separa el contenido sustancial de tutoriales y referencias; etiqueta una sección cuando cualquiera de las partes sea pequeña.

Antes de escribir un tutorial, clasifica en privado el conocimiento inicial del lector y cada concepto como principiante, intermedio o avanzado. Establece los prerrequisitos antes que los conceptos dependientes, aumenta la dificultad gradualmente y traslada el material avanzado innecesario a un tutorial o referencia posterior.

Redacta en este orden: localiza el documento en el árbol; fija su detalle permitido; elige tutorial o referencia; para un tutorial, ordena los conceptos por prerrequisito y dificultad; reubica el detalle propiedad de los descendientes; sustituye las explicaciones de nivel inferior por enlaces a sus propietarios.

## La taxonomía de niveles: un hogar por hecho

Cada hecho tiene un hogar: el nivel cuyo trabajo es ese hecho; en cualquier otro sitio, enlaza allí.

| Nivel | Función | NO pertenece allí |
|---|---|---|
| `AGENTS.md` raíz | Órdenes permanentes: reglas que un agent necesita en contexto en cada sesión, de una a tres líneas cada una, enlazando a su hogar | Historias, ejemplos trabajados, procedimientos situacionales, cualquier cosa reformulada desde un hogar enlazado |
| `AGENTS.md` de subárbol (`packages/`, `examples/`, `docs/`, `.agents/notes/`) | Órdenes específicas de ese subárbol | Reglas de todo el repo que el archivo raíz ya lleva |
| [architecture.md](architecture.es.md) | Mapa ordenado: composición, paquetes core, loop, seams, puntos de extensión; léelo antes de cambiar `packages/` | Definiciones de tipos (→ subsystems), detalle por paquete (→ package READMEs), justificación de decisiones (→ Agent Notes), anotaciones de estado de implementación |
| [subsystems/](subsystems/README.es.md) | Una página de referencia por subsystem: definiciones de tipos, semántica y la API generada de Cordis | Narración de comportamiento (→ architecture.md) |
| [Agent Notes](../.agents/notes/README.es.md) | Registros de decisión activos: el porqué, lo que se renunció y la verificación requerida; las notas de `implemented/` describen la realidad publicada en presente | Planes de migración, checklists de tareas de aceptación, recorridos de fixtures y lenguaje de especificación («should…») una vez que la decisión se ha publicado; las notas archivadas son historia congelada, nunca autoridad vigente |
| [postmortem/](postmortem/README.es.md) | Historias de incidentes — el único nivel donde encaja la narrativa de batallas | — |
| [cookbook/](cookbook/adding-a-package.es.md) | Guías paso a paso con pasos de verificación numerados | Justificación de diseño (→ el Agent Note que enlaza cada guía) |
| [user/](user/index.es.md) | Guías orientadas al producto publicadas por el sitio web de documentación | Tablas de referencia generadas, procedimientos de contribución, historial de decisiones |
| Package README | El contrato por paquete: config, semántica, limitaciones, puntos de extensión y [Model Experience](cookbook/adding-a-package.es.md#4-write-the-package-readme) | Reformulación de JSDoc, reformulación de catálogos generados (tablas de eventos/herramientas), asuntos de otros paquetes |
| [development.md](development.es.md) | Configuración del contribuidor, flujo de trabajo diario y un resumen de CI; un par bilingüe bajo el [contrato i18n](i18n/README.es.md) | Justificación de runtime/versiones (→ Agent Notes), listas comprobación a comprobación que se desvían de los scripts de `package.json` |
| Referencia generada: las regiones `cordis-surface` por página en [subsystems/](subsystems/README.es.md), el [Cordis core API + nivel heredado](cordis-api/context.es.md), [tool-catalog](tool-catalog.es.md), [config-catalog](config-catalog.es.md), [persistence-catalog](persistence-catalog.es.md), [module-graph.md](module-graph.es.md) | Fuentes en inglés exhaustivas regeneradas desde el código fuente y controladas por frescura; las contrapartes chinas revisadas siguen el [flujo de trabajo de emparejamiento](i18n/README.es.md#scope-and-exclusions) | Ediciones a mano de fuentes o regiones generadas en inglés; las contrapartes chinas solo se actualizan mediante el emparejamiento |
| Skills (`.agents/skills/`) | Flujos de trabajo reutilizables y estándares de decisión especializados | Contratos de producto y runtime (→ docs o código fuente) |

Ubicación: bugs → post-mortems; justificación → Agent Notes; procedimientos → recetarios (cookbooks); definiciones de tipos → subsystems; contratos de paquetes → READMEs; órdenes permanentes → `AGENTS.md` raíz con un enlace de justificación.

## Reglas de redacción

- **Documenta el estado actual, no el historial de cambios.** Evita «previously/now/no longer», PRs, commits y posiciones de pila en la prosa duradera; nombra el mecanismo vivo. Pon las historias de cambio en commits, PRs, Agent Notes o post-mortems; estos dos últimos pueden citar PRs fusionados e issues como evidencia.
- **Todo cambio no trivial incluye al menos un Agent Note en el mismo PR.** Actualiza la nota propietaria o añade una; solo las ediciones mecánicas o locales están exentas ([alcance](../.agents/notes/README.es.md#when-to-write-one)).
- **Una línea física por párrafo** (`verify-md-wrap`): usa el soft-wrap del editor. Los bloques de código, las tablas y la estructura de listas conservan su formato; los comentarios de código se mantienen bajo el límite de columnas del linter.
- **Los bloques `ts` delimitados deben compilar** (`doc-typecheck`); una declaración de tipo pegada y su JSDoc original usan ` ```ts type-equiv `, mientras que una declaración de clase pública sin cuerpo usa ` ```ts public-api `; registra cualquiera de las dos en el manifest para que ninguna pueda desviarse ([mecánica](development.es.md#documenting-types-verbatim-ts-type-equiv)).
- **La [página de subsystems](subsystems/README.es.md) propietaria se actualiza en el mismo cambio** que remodela un tipo documentado. `verify-type-equiv` detecta pegadas desviadas, no tipos nuevos nunca documentados; un tipo se documenta en la página del grupo de paquetes que lo declara ([alcance de páginas](../.agents/notes/implemented/process/2026-08-03-package-anchored-subsystem-pages.es.md)).
- **Los pares se actualizan juntos**: el trabajo de agent activo en una sola pasada, [guiado por la terminología](i18n/terminology.es.md), recoloca las anotaciones de primer uso, preserva la prosa intacta y vuelve a registrar; `dsh-translate-docs` sigue siendo invocado por el usuario ([contrato](i18n/README.es.md)).
- **Los comentarios y el JSDoc declaran contratos completos, no transcript (transcripción) de razonamientos.** Conserva el comportamiento, los fallos, la temporización, la propiedad, la modalidad, las excepciones, las consecuencias y la orientación no obvia; elimina la narración, los recorridos de tests, el análisis de revisión y la reformulación de código. Mantén el contrato local y enlaza su justificación. Usa [dsh-prose-standard](../.agents/skills/dsh-prose-standard/SKILL.es.md) para los detalles.
- Escribe de forma directa: nombra actores y hechos ([decisión](../.agents/notes/implemented/process/2026-08-09-concrete-prose-names-actors-and-recorded-facts.es.md)). Reserva `seam` para la capacidad definida. Nombra la comprobación, el tipo, la API, la operación o el comportamiento exactos en lugar de «gate», «vocabulary» o «surface» metafóricos.

## Presupuestos de palabras

[scripts/doc-budgets.manifest.json](../scripts/doc-budgets.manifest.json) fija los techos de los documentos permanentes; `pnpm run verify-doc-budgets` rechaza archivos en exceso o ausentes.

Cuando el gate se pone en rojo:

1. **Reubica** el contenido que pertenece a otro nivel; deja un enlace de una línea si hace falta.
2. **Condensa** el contenido que pertenece aquí pero puede ser más corto.
3. **Sube** el techo solo cuando las palabras necesiten el espacio; justifica el diff del manifest en el PR. Un techo demasiado bajo es un bug del presupuesto.

Los techos son barandillas, no objetivos de reducción. En el objetivo o por debajo de él, conserva al menos un 5 % de margen; por encima del objetivo, congela el techo hasta que la reubicación o la condensación dejen el documento bajo el objetivo. Baja un techo solo cuando el documento todavía tenga espacio, y súbelo cuando el contenido se borraría en caso contrario. Objetivos: `AGENTS.md` raíz ≤ 1,600 palabras; `architecture.md` ≤ 1,800; `AGENTS.md` de subárbol ≤ 600, excepto `packages/AGENTS.md` ≤ 650 y este archivo ≤ 1,250; `packages/README.md` ≤ 600. La revisión gobierna los niveles sin presupuesto.

## La checklist de slop

Busca estos problemas en cualquier documento; [dsh-doc-standards](../.agents/skills/dsh-doc-standards/SKILL.es.md) ejecuta esta lista como auditoría:

- La misma regla declarada en más de un hogar. Haz grep de una frase distintiva; conserva un hogar y enlaza el resto.
- Historia narrada o historias de guerra: «previously», «now», «no longer», «used to», «renamed», «was moved», PRs o commits. Declara el hecho actual; enlaza un Agent Note o un post-mortem cuando haga falta.
- Anotaciones de estado de implementación en prosa o diagramas («implemented!», «future: …»). El estado se pudre; la estructura del repo y los manifests de paquetes lo llevan.
- Catálogos, JSDoc o inventarios de tests, paquetes y estado reformulados a mano cuando el código fuente o un generador son la autoridad.
- Transcript (transcripción) de razonamientos: narración de implementación paso a paso, demostración de ramas obvias, recorridos de tests o alternativas locales rechazadas. Conserva el contrato resultante o la justificación durable; elimina el camino usado para derivarlo.
- Justificación repetida junto a métodos hermanos en lugar de una sola vez en la capacidad o el helper propietario.
- Muros de párrafo: un párrafo que carga varias reglas e incisos entre paréntesis. Divídelo o degrada el detalle a su hogar.
- Inflación de énfasis: negrita, MAYÚSCULAS o «critically» en todas partes hace que nada destaque. Reserva el énfasis para la cláusula que cambia el comportamiento.
- Lenguaje de especificación en los Agent Notes de `implemented/`: «should», planes de migración, checklists de aceptación. Un Agent Note implementado describe lo que es, según las [instrucciones de notas implementadas](../.agents/notes/implemented/AGENTS.es.md).

## Haz referencias cruzadas con enlaces comprobables por máquina, nunca prosa libre

Enlaza las referencias del repositorio con rutas Markdown relativas, nunca con nombres de archivo sueltos ni números de Agent Note. `verify-md-links` rechaza destinos ausentes y anclas `#fragment` muertas ([justificación](../.agents/notes/implemented/process/2026-06-18-markdown-cross-link-lint.es.md)).
