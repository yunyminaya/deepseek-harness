---
name: dsh-doc-standards
description: Úsalo al redactar, mover, revisar o auditar documentación en el repo deepseek-harness — para elegir jerarquía y nivel de detalle, separar tutoriales de referencias, comprobar la progresión de los tutoriales, recortar doc slop, responder a un fallo de verify-doc-budgets o a peticiones como "mejora la documentación", "audita la documentación", "dónde debería documentarse esto" o "este documento es demasiado largo".
---

# Aplicar el estándar de documentación de DeepSeek Harness

[English](SKILL.md) | Español

Las reglas de documentación viven en [docs/AGENTS.md](../../../docs/AGENTS.es.md).
Este flujo cubre ubicación, auditorías del corpus, budgets y validación en Markdown, JSDoc y comentarios de código.
Es guía, no script; usa [dsh-prose-standard](../dsh-prose-standard/SKILL.es.md) para cobertura obligatoria y criterio editorial, y nunca trates la longitud por sí sola como un defecto.

## Fuentes de verdad (léelas, no las vuelvas a resumir)

- [docs/AGENTS.md](../../../docs/AGENTS.es.md) — jerarquía, formas tutorial/referencia, taxonomía, budgets y checklist de slop.
- [.agents/notes/README.md](../../notes/README.es.md) — cuándo una decisión merece una Agent Note, cómo archivarla y qué va dentro (bloque de cabecera, esqueleto por ciclo de vida y mandato de Alternatives considered, protegido por `verify-agent-note-format`); [docs/postmortem/README.md](../../../docs/postmortem/README.es.md) — cuándo un incidente merece un post-mortem.
- [docs/i18n/README.md](../../../docs/i18n/README.es.md) — reglas de emparejamiento bilingüe; editar cualquiera de los lados obliga a actualizar la contraparte en el mismo cambio.
- [AGENTS.md](../../../AGENTS.md) raíz — las órdenes permanentes cuya disciplina de presupuesto protege este skill.
- [Archived Agent Notes](../../notes/archived/AGENTS.md) — instantáneas históricas congeladas, excluidas del mantenimiento editorial y de los gates evolutivos de documentación.

## Revisa la estructura antes de la prosa

Aplica el orden de autoría del estándar a todo documento orientado a humanos dentro del alcance.
No apliques este pase estructural a Agent Notes.
Clasifica un post-mortem como una referencia acotada a un incidente; conserva su evidencia cronológica obligatoria sin tratar la cronología como una secuencia didáctica.

1. Localiza el documento en los árboles del repositorio y de navegación.
Declara su propio tema e identifica a sus hijos directos.
2. Fija el nivel de detalle permitido.
Conserva el detalle completo sobre el tema del documento, resume a los hijos directos por propósito, responsabilidad y comportamiento de alto nivel, y mueve explicaciones más profundas a sus descendientes propietarios con enlaces.
Trata la infraestructura de pruebas como propiedad de descendientes salvo que sea el tema del documento.
3. Clasifica el documento por su uso previsto, no por su ruta o título.
Un tutorial debe conducir trabajo ordenado hacia un resultado observable; una referencia debe servir de consulta dentro de un alcance explícito sin exigir lectura secuencial.
4. Para un tutorial, clasifica en privado al lector inicial y a los conceptos como principiante, intermedio o avanzado.
Traza cada concepto hasta sus prerrequisitos, reordena material prematuro y mueve detalle avanzado opcional a un tutorial o referencia posterior.
5. Separa las formas mixtas sustanciales.
Pon una forma secundaria pequeña en una sección claramente etiquetada.

Luego comprueba restricciones que vuelven costosa o incorrecta la ubicación:

- Los documentos emparejados (`pnpm run verify-translation-pairing --list`) obligan a actualizar la contraparte zh y a volver a registrar con `--write` en cada edición; prefiere un hogar no emparejado para contenido que vaya a cambiar con frecuencia.
- Los catálogos generados nunca se editan a mano; si el hecho pertenece allí, cambia la fuente del generador.
- Antes de renombrar o mover cualquier doc, busca referencias entrantes: `verify-md-links` detecta destinos de enlaces Markdown y anclas `#fragment` sobre archivos Markdown (slugs de encabezado y `<a id>` explícitos), y `verify-doc-refs` detecta citas a `docs/*.md` en comentarios TypeScript; las anclas citadas desde strings TypeScript aún necesitan un grep manual cuando su salida nunca llega a Markdown cubierto por gates.
- Un movimiento es atómico: elimina del hogar viejo, añade al nuevo y corrige todos los enlaces entrantes en el mismo cambio.

## Audita el corpus

Tras el pase estructural, persigue la checklist de slop del estándar con las sondas más baratas primero.
Verifica y trae la base viva del PR y luego ejecuta `pnpm --silent run change-scope --base <verified-base-ref>` para identificar rutas commiteadas y sucias antes de aplicar criterio semántico.
Tras un retarget o un merge con la base, vuelve a ejecutar el informe y audita la prosa introducida por la nueva base.

1. Mide: `pnpm run verify-doc-budgets --list`, luego `git ls-files '*.md' ':(exclude)vendor/**' | xargs wc -w | sort -rn | head -30` para detectar valores atípicos sin budget.
2. Busca fugas de reasoning transcript — historia narrada, citas muertas de sesiones de diseño, coreografía de review, narración de flujo de control, walkthroughs de tests — con [dsh-trim-cot-leakage](../dsh-trim-cot-leakage/SKILL.es.md), que define la taxonomía, las baterías de recall y las reglas sobre qué conservar o eliminar.
Conserva solo un contrato no obvio o una justificación duradera; la misma justificación repetida junto a métodos hermanos conserva un solo hogar.
3. Busca duplicación con grep de frases distintivas.
Conserva un solo hogar y sustituye las otras copias por enlaces.
4. Sustituye catálogos escritos a mano, inventarios de test/estado y reformulaciones de JSDoc por el árbol autoritativo, el script o la referencia generada.
5. En Agent Notes bajo `implemented/`, elimina planes de migración, listas de tareas de aceptación y lenguaje especulativo en futuro.
Conserva contratos de verificación concisos que identifiquen los comportamientos y tiers que fijan la decisión enviada, además de lagunas de cobertura con nombre.
6. Si eliminar prosa cambia un comportamiento prometido en vez de su explicación, usa primero una Agent Note propuesta (sigue [dsh-find-simplifications](../dsh-find-simplifications/SKILL.es.md)).

Excluye `.agents/notes/archived/` de auditorías y ediciones del corpus.
La prosa activa puede reparar, redirigir o eliminar un enlace entrante, pero nunca seguir una limpieza global hacia el destino congelado.

Conserva cada regla que soporte carga, preferiblemente en una a tres líneas más un enlace a su justificación.
Recorta historias, duplicados, notas de estado y el camino usado para derivar la regla.
No crees una explicación nueva solo para reubicar razonamiento descartable.

## Cuando verify-doc-budgets se pone en rojo

Aplica la política ordenada relocate-condense-raise de [docs/AGENTS.md](../../../docs/AGENTS.es.md); este skill solo aporta las sondas del flujo anteriores.

## Validación e higiene de PR

Ejecuta al menos `pnpm run doc-sync`, `pnpm run lint` y `git diff --check`; los cambios de JSDoc pueden regenerar catálogos.
Si cambió un documento emparejado, sigue la [ruta rutinaria ligera](../../../docs/AGENTS.es.md#writing-rules) y ejecuta `pnpm run verify-translation-pairing --write <pair>`.
El cuerpo del PR debe incluir deltas de palabras, explicar cualquier excepción deliberadamente larga y listar las comprobaciones.
