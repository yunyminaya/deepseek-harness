# Agent Notes

[English](README.md) | Español

Un tipo de documento de diseño vive aquí. Un **Agent Note** registra una decisión o propuesta que afecta este código — el *por qué* y *a lo que renunciamos*, las partes que el código y los documentos no pueden transportar. Este archivo define dónde viven los Agent Notes, cuándo escribir uno y [el formato dentro del archivo](#the-file-format).

## Disposición y nomenclatura

Cada Agent Note tiene dos ejes, ambos codificados en su **ruta** — `{lifecycle}/{class}/yyyy-mm-dd-topic-title.md`:

- **Lifecycle** (la carpeta de nivel superior) es el estado del Agent Note, y un Agent Note se mueve entre carpetas a medida que ese estado cambia:
  - **`proposed/`** — propuestas revisadas antes de la implementación; aún no construidas (o solo parcialmente).
  - **`implemented/`** — la decisión se envió. El archivo registra lo que se decidió y lo que se rechazó, y se **mantiene actualizado con lo que realmente se envió**: cuando el código luego mueve un archivo, cambia el nombre de un paquete o modifica una clave/valor predeterminado, el Agent Note se actualiza en el mismo cambio para que coincida (solo hechos — rutas, nombres, estructura — no la decisión misma). Ver [implemented/AGENTS.md](implemented/AGENTS.es.md).
  - **`rejected/`** — la propuesta se consideró y se rechazó. Guárdala solo mientras su justificación evita un error tentador y significativo; de lo contrario, elimina la tríada completa.
- **Class** (la carpeta anidada) es el *tipo* de decisión — ver [Clasificación](#classification) a continuación.

La fecha en el nombre de archivo es cuando el tema se **propuso por primera vez** (según el historial de git). Las referencias cruzadas entre Agent Notes usan enlaces markdown relativos (`[topic](../../implemented/architecture/2026-…-….es.md)`) — nunca prosa en bruto o números — para que sean verificables mecánicamente y sobrevivan a los movimientos entre carpetas.

El árbol de ciclo de vida activo es el inventario de trabajo: navega sus carpetas de ciclo de vida/clase o busca en el repositorio. No agregues un `INDEX.md` centralizado; el [Agent Note sin índice](implemented/process/2026-07-19-remove-generated-agent-note-index.es.md) posee la justificación. Los registros implementados de bajo valor futuro se mueven al árbol congelado separado [`archived/`](archived/AGENTS.es.md) que se describe a continuación.

## Clasificación

Cada Agent Note pertenece a una clase codificada en ruta del conjunto cerrado en `scripts/agent-note-tree.ts`; el gate de clasificación rechaza otras carpetas. Agregar una clase requiere actualizar el conjunto canónico y esta sección. Ver el [Agent Note de clasificación](implemented/process/2026-06-20-agent-note-classification.es.md).

| Class | Lo que cubre |
|---|---|
| `feature` | Una nueva funcionalidad orientada al usuario o al modelo. |
| `bug-fix` | Corrige un defecto o cierra una brecha que un post-mortem reveló. |
| `simplification` | Elimina código, comportamiento o área de superficie sin agregar funcionalidad. |
| `architecture` | Una decisión estructural sobre el **fuente enviado** — cómo se relacionan los paquetes, cuál es el vocabulario en tiempo de ejecución. |
| `process` | Herramientas, políticas o flujos de trabajo **alrededor** del código — gates, el gestor de paquetes, vendoring — no el comportamiento en tiempo de ejecución. |
| `testing` | Infraestructura y estrategia de pruebas. |

La línea `architecture` / `process`: **architecture** trata sobre el código fuente que enviamos; **process** trata sobre las herramientas y flujos de trabajo circundantes. (`refactor` está deliberadamente ausente — se superpone con `simplification`, cuyo discriminador, "¿cambia el comportamiento observable?", ya lo cubre).

## Archivo y eliminación

Archiva un Agent Note implementado cuando la decisión enviada está completa y su justificación es poco probable que guíe el trabajo futuro. Mantenlo activo cuando sus alternativas, límite de propiedad, garantía negativa, semánticas duraderas o de cable, regla de seguridad o condición de reintroducción siguen siendo útiles. Nunca archives una nota propuesta: rechaza una propuesta obsoleta. Mantén una nota rechazada solo mientras evita un error plausible; de lo contrario, elimina sus archivos inglés, chino y sidecar juntos. Usa el flujo de trabajo calibrado [`dsh-archive-agent-notes`](../skills/dsh-archive-agent-notes/SKILL.md) en lugar del recuento de palabras, la edad o una cuota objetivo.

El archivo está codificado en ruta como `archived/{class}/yyyy-mm-dd-topic-title.md`; `implemented` está deliberadamente ausente porque solo las notas implementadas pueden entrar en él. Un cambio de archivo mueve la tríada completa inglés/chino/sidecar, conserva `Status: implemented`, inserta la misma línea `Archived: YYYY-MM-DD` inmediatamente debajo de ese estado en ambos archivos de idioma, regraba el sidecar y repara o elimina los enlaces entrantes. Estos son los únicos cambios de contenido permitidos durante el archivo.

Una vez sellada, cada tríada archivada está permanentemente congelada. No la edites, traduzcas, reformates, actualices, muevas o elimines, y no la trates como autoridad para el comportamiento actual. Los gates de documentación omiten fuentes archivadas, incluidos sus enlaces salientes; la prosa activa aún puede enlazar a una nota archivada cuando cita intencionalmente la historia. [`verify-archived-agent-notes`](../../scripts/verify-archived-agent-notes.ts) aplica el árbol de clases cerrado, tríadas completas, metadatos de archivo, hashes de sidecar y el manifiesto de contenido congelado de solo agregado. El [Agent Note de política de archivo](implemented/process/2026-07-26-frozen-agent-note-archive.es.md) posee la justificación.

## Cuándo escribir uno

Cada cambio no trivial DEBE agregar o actualizar al menos un Agent Note en el mismo PR. Un cambio es no trivial cuando altera el comportamiento, la arquitectura, un contrato compartido entre archivos o paquetes, el proceso o las herramientas, la estrategia de pruebas, un formato en disco, de cable o de configuración, u otra decisión que un mantenedor pueda razonablemente revisitar. Una propuesta para un trabajo futuro sustancial comienza en `proposed/`; una decisión ya tomada comienza en `implemented/`. Elige la carpeta de clase que coincida con la decisión (ver [Clasificación](#classification)).

Actualizar el Agent Note que ya posee la decisión satisface la regla; no crees un duplicado. Solo una edición puramente mecánica o local sin cambio en el comportamiento, contratos, estructura, proceso o justificación está exenta. Un Agent Note nunca se edita en una *decisión diferente*: reemplázalo con uno nuevo y mantén ambas notas con enlaces cruzados a menos que la nota antigua se consolide completamente bajo la regla a continuación. Editar un Agent Note `implemented/` para rastrear dónde vive su decisión existente es requerido, no prohibido; ver [implemented/AGENTS.md](implemented/AGENTS.es.md).

Un Agent Note implementado que está completamente reemplazado puede consolidarse en la nota propietaria actual y eliminarse. Antes de la eliminación, el propietario debe conservar cada justificación única, alternativa, consecuencia, verificación requerida y brecha de cobertura nombrada; reparar cada enlace entrante; y eliminar la contraparte china y el registro de consistencia en el mismo cambio. El reemplazo parcial no califica: mantén ambas notas con enlaces cruzados y actualiza cada hecho que permanezca actual. La consolidación no debe reescribir el archivo antiguo en su opuesto ni confiar en el historial de git como la única copia de la justificación.

Una nota de adición de funcionalidad puede consolidarse en la nota de eliminación posterior solo cuando la funcionalidad está ausente del código de producción, configuración, esquemas, formatos duraderos o de cable, migración y comportamiento de compatibilidad; ningún documento actual la presenta como disponible; ninguna prueba la ejerce como comportamiento admitido. La justificación de eliminación y las pruebas que verifican la ausencia pueden permanecer. El propietario de eliminación conserva la motivación original, por qué ya no justifica la funcionalidad, alternativas a la eliminación completa, la capacidad a la que se renunció, condiciones de reintroducción y verificación de ausencia completa. Los inventarios de implementación obsoletos y las pruebas que solo verificaban el comportamiento eliminado no son evidencia de verificación actual. Eliminar un transporte, valor predeterminado, implementación o presentación es un reemplazo parcial, al igual que cualquier dato duradero o manejo de compatibilidad sobreviviente.

## El formato de archivo

Cada Agent Note activo sigue un formato dentro del archivo, aplicado por `pnpm run verify-agent-note-format` ([scripts/verify-agent-note-format.ts](../../scripts/verify-agent-note-format.ts), parte de `doc-sync`); la justificación para el formato — y las alternativas que rechazó — es [el Agent Note de formato uniforme](implemented/process/2026-07-05-uniform-agent-note-format.es.md). Las notas archivadas retienen el formato que tenían cuando se sellaron más la línea de fecha de archivo anterior.

### El bloque de encabezado

Las primeras tres líneas de cada Agent Note son exactamente:

```markdown
# Agent Note: <title>

Status: <status>
```

seguidas de una línea en blanco. El valor de `Status:` es una de tres formas y debe coincidir con la carpeta de ciclo de vida donde se encuentra el archivo — el gate las verifica:

- `Status: proposed`
- `Status: implemented`
- `Status: rejected — <why, in one line>`

El estado no lleva fechas ni paréntesis: el nombre de archivo contiene la fecha de propuesta inicial, git contiene todo lo demás, y una nota "aceptada en forma enmendada" es contenido del cuerpo (establece la enmienda donde se declara la decisión). El motivo de rechazo es el único estado con contenido, porque el veredicto de un Agent Note rechazado es el hecho que los lectores buscan.

### El esqueleto del cuerpo

Cada Agent Note abre su cuerpo con `## Problem` — la motivación, escrita para existir sin la solución. Lo que sigue depende del ciclo de vida; las secciones recurrentes usan estos nombres canónicos y nada más, mientras que las secciones técnicas verdaderamente personalizadas (topología de paquetes, contratos de cable, esquemas) permanecen de forma libre entre las requeridas.

#### `proposed/`

```markdown
## Problem
## Proposal
…secciones personalizadas…
## Alternatives considered
## Acceptance criteria
## Risks
```

`## Proposal` es el cambio previsto y legítimamente puede hablar en tiempo futuro — planes, pasos de migración y preguntas abiertas pertenecen aquí mientras el trabajo no está construido. `## Acceptance criteria` dice qué estado observable significa hecho. `## Risks` cubre tanto lo que podría salir mal como lo que el cambio conscientemente renuncia.

#### `implemented/`

```markdown
## Problem
## Decision
…secciones personalizadas…
## Alternatives considered
## Consequences
```

`## Decision` describe la realidad enviada en tiempo presente, y todo el archivo se mantiene actualizado con él según [implemented/AGENTS.md](implemented/AGENTS.es.md). `## Consequences` registra lo que el compromiso de intercambio **y** compró. Los encabezados de época de propuesta son lenguaje de especificación aquí y el gate los rechaza: `## Proposal`, `## Plan`, `## Migration plan` y `## Acceptance criteria` no pueden aparecer en un Agent Note implementado (la [lista de verificación de slop](../../docs/AGENTS.es.md) explica por qué). Una sección `## Testing`, `## Deferred` o `## Related` es aceptable donde establece un hecho en tiempo presente.

#### `rejected/`

Un Agent Note rechazado es la propuesta, congelada: mantiene las secciones que tenía en época de propuesta (incluido `## Acceptance criteria` o `## Plan`), y el veredicto vive en la línea `Status:`. Solo se aplican el bloque de encabezado, el abridor `## Problem`, una sección `## Proposal` y el mandato de alternativas consideradas a continuación.

### Alternatives considered — obligatorio

Cada Agent Note lleva una sección `## Alternatives considered`: cada alternativa genuina y por qué perdió, un párrafo liderado en negrita por alternativa o una subsección `### Why not <X>?` por cada una disputada. Una decisión registrada sin lo que venció invita a la re-litigación — el fracaso que las Agent Notes existen para prevenir.

Las alternativas se registran, nunca se inventan. Un Agent Note fechado antes de 2026-07-05 cuyas alternativas no son reconstruibles desde el registro lleva este comentario exacto en lugar de la sección, que el gate acepta solo para archivos previos al formato:

```markdown
<!-- agent-note-format: alternatives-not-recorded (pre-format Agent Note) -->
```

### Moverse entre ciclos de vida

Mover un archivo entre carpetas de ciclo de vida significa actualizar la línea `Status:` y volver a satisfacer el esqueleto de esa carpeta en el mismo cambio — el gate falla el movimiento de lo contrario. Concretamente, `proposed/` → `implemented/` reescribe `## Proposal` en un `## Decision` en tiempo presente, pliega `## Acceptance criteria` y `## Risks` en `## Consequences` (o una sección `## Testing`/`## Verification` en tiempo presente para lo que ahora fija el comportamiento) y descarta planes a favor de lo que se envió — la reescritura que [implemented/AGENTS.md](implemented/AGENTS.es.md) requiere, hecha mecánica. `proposed/` → `rejected/` solo agrega el motivo a la línea `Status:` y congela el archivo.

### Contrapartes chinas

Una contraparte `.zh.md` refleja la estructura de su hermano inglés sección por sección bajo el [contrato i18n](../../docs/i18n/README.es.md); los tokens de encabezado verificados mecánicamente (`# Agent Note: ` y la línea `Status:`) permanecen en inglés verbatim. El gate de formato omite los archivos `.zh.md` — el gate de emparejamiento verifica su consistencia.