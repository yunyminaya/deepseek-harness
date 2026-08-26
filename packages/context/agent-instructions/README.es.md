# @deepseek-ai/dsh-agent-instructions

[English](README.md) | Español

Carga de instrucciones de workspace por sesión para archivos compatibles con `AGENTS.md`. El plugin inyecta la cadena inicial de instrucciones globales de usuario y de proyecto en el historial duradero, luego descubre archivos anidados y reporta cambios o eliminaciones posteriores tras llamadas exitosas a herramientas del sistema de archivos.

## Ciclo de vida

El primer `agent/pre-step` elegible de cada sesión viva compone la línea base. Cuando la decisión descendente entra en un primer lote de pasos no vacío, el plugin pliega la línea base en ese lote final justo después del prompt reclamado, de modo que el prompt directo y la línea base duradera entran en el paso 1 y llegan juntos a la primera petición. Una decisión de primer paso rechazada o vacía deja la línea base en la bandeja de entrada `next-step` del agent para un despertar posterior. El loader lee `$DSH_HOME/AGENTS.md` y, después, en cada directorio desde la raíz del proyecto hasta `agent.session.header.cwd`, cada candidato base existente y después cada candidato de overlay local existente. Dentro de un mismo directorio, los candidatos cuyo contenido es byte-idéntico tras recortar el espacio en blanco inicial y final se colapsan al candidato más temprano en el orden configurado, por lo que un `CLAUDE.md` que meramente duplica a su hermano `AGENTS.md` se renderiza una sola vez. Si un contexto de workspace previamente encolado sigue pendiente, el plugin elimina y reemplaza ese elemento exacto de la bandeja de entrada en lugar de acumular duplicados. Una sesión reanudada conserva una línea base visible compatible y añade solo transiciones de archivos actuales; un descubrimiento, precedencia, raíz de proyecto o identidad de presupuesto cambiados pliega en su lugar una línea base completa explícitamente superadora en el lote entrante.

El plugin observa también los resultados inmutables de `tools/result` para llamadas exitosas de primera parte a `read`, `write` y `edit`. Cada toque aceptado comprueba los ámbitos descendientes recién alcanzados y todos los ámbitos cargados previamente. Cada nombre de candidato configurado es un ámbito independiente en su directorio: un archivo recién presente encola una adición en la bandeja de entrada del agent; un archivo cambiado encola un reemplazo; un archivo que desaparece o se convierte en duplicado por directorio de un candidato anterior encola un aviso de eliminación. Las llamadas nativas y los sub-desplazamientos de Code Mode comparten esta vía: los toques anidados burbujean a través de los tokens de ejecución opacos del padre hasta que se asienta el resultado de nivel superior, y los toques producidos dentro de un paso del agent loop no inician su proyección asíncrona hasta el `step/end` duradero. Las ejecuciones directas de herramientas fuera de un paso abierto se proyectan de inmediato. Esto preserva la adyacencia llamada/resultado/paso de herramienta sin depender de la sincronización del sistema de archivos. El descubrimiento sigue la actividad estructurada del sistema de archivos en lugar del `cd` de shell, porque cada llamada local de bash inicia un shell nuevo y analizar sintaxis arbitraria de shell no sería fiable.

Las lecturas de instrucciones usan el provider opcional `ctx.fs`. El plugin no inyecta `fs` estáticamente, por lo que los árboles de producto sin provider siguen arrancando y la carga de instrucciones se convierte en un no-op hasta que haya un provider presente. Resuelve cada candidato y hace stat del resultado, de modo que un symlink como componente final se sigue hasta su destino: un enlace a un archivo regular carga el contenido de ese destino, mientras que una ruta ausente o un destino que no es archivo (incluido un enlace a un directorio) es una ausencia confirmada. Una excepción de resolve o stat marca en su lugar el ámbito de ese candidato como temporalmente no disponible. La cancelación por prefijo y la cancelación dinámica de herramientas se propagan a través de la resolución, las sondas de metadatos y las lecturas en streaming. Un fallo de provider tras cargar un archivo se trata como temporalmente no disponible, no como prueba de que el archivo se eliminó.

## Forma del prompt

Las instrucciones de la línea base son mensajes duraderos de rol de usuario enmarcados con el patrón familiar de recordatorio de sistema:

```md
<system-reminder>
The following workspace instructions may be relevant to your work. Use them as guidance when applicable. More specific instructions take precedence over broader ones. They do not override system, developer, or direct user instructions.

Instructions from: ~/.dsh/AGENTS.md

...

Instructions from: AGENTS.md

...
</system-reminder>
```

Los ámbitos recién alcanzados usan un `user/message` duradero con fuente:

```md
<system-reminder>
Additional instructions from: packages/app/AGENTS.md

These instructions apply to work under `packages/app`. Use them as guidance when relevant; more specific instructions take precedence. They do not override system, developer, or direct user instructions.

...
</system-reminder>
```

Una edición del mismo archivo empieza con `Updated instructions from: <path>` y dice que se use el contenido nuevo en lugar del contenido cargado previamente. Cuando un candidato desaparece o se convierte en duplicado por directorio de un candidato anterior, el mensaje es `Instructions removed: <path>` seguido de `The previously loaded instructions from this file no longer apply.` El texto literal `</system-reminder>` en cualquier parte del contenido de instrucciones o de los metadatos de ruta, ámbito y presupuesto visibles para el modelo se escapa para que el texto controlado por el repositorio no pueda cerrar el marco propiedad del plugin.

El plugin es dueño del enmarcado completo `<system-reminder>`, y cada `user/message` inyectado llega al modelo verbatim sin envoltorio del núcleo.

## Estado y actualización

El texto visible para el modelo no contiene marcadores de estado ocultos. Cada evento de línea base o de contexto dinámico lleva en su lugar una fuente tipada `agent-instructions` con una lista de cambios `{ action, scope, path, digest? }`; una línea base completa lleva también `baseline: true` y un `baselineIdentity` derivado del descubrimiento, la precedencia, la raíz de proyecto y la configuración de presupuesto normalizados. Un `user/message` duradero coincidente confirma una línea base encolada y sus versiones de candidato. Un pre-step entrante espera a cada proyección encolada, pliega el contexto recién compuesto en su lote final inmediatamente después de los mensajes reclamados y elimina la copia pendiente de la bandeja de entrada; el rechazo mantiene encolado el contexto actual. Si un listener reescribe un mensaje de workspace reclamado sin introducir su reemplazo, un límite posterior recomponne el contexto actual. Los resultados anidados agregan los toques de archivo exitosos bajo su token de ejecución padre, incluso cuando un resultado compuesto posterior queda bloqueado; el resultado de nivel superior transfiere esos toques al paso de sesión abierto en ese momento o directamente a la cola de proyección por agent. Un `step/end` libera sus toques escalonados solo después de que ese límite esté en el historial duradero, y las proyecciones serializadas se reconcilian contra los eventos de sesión visibles más la bandeja de entrada actual antes de reemplazar el único contexto de workspace pendiente.

Una ruta y un digest de contenido SHA-1 sin cambios no se vuelven a inyectar. Una caché de provider por sesión y por ámbito almacena solo `{ path, version, digest, trimmedDigest }`: cuando el `FsVersion` opaco del provider y el estado visible efectivo coinciden, la reconciliación se salta la lectura de contenido; una versión cambiada dispara una lectura acotada y una confirmación SHA-1 antes de cualquier actualización visible para el modelo. El `trimmedDigest` — SHA-1 sobre el contenido con espacios recortados — es la clave de duplicado por directorio, por lo que un archivo sin cambios puede eliminarse cuando un candidato anterior converge en su contenido. La reanudación funciona porque el estado SHA-1 persiste en la fuente tipada, mientras que una caché de versiones en memoria vacía provoca solo una lectura de confirmación. La compactación rearma un ámbito después de que su evento de contexto abandona la superficie visible incluso cuando la versión en caché no cambió. Una eliminación es una lápida, por lo que una reaparición posterior de un candidato se vuelve a cargar. Un cambio visible para el modelo entra en la fuente, el estado pendiente y la caché de versiones solo cuando su sección específica de archivo conserva al menos un byte de contenido, o cuando su contenido original es genuinamente vacío. El truncamiento parcial registra el digest del contenido completo cuando sobrevive cualquier byte de contenido; el truncamiento a cero sigue siendo elegible para un toque posterior, mientras que una actualización de versión con el mismo digest actualiza solo la caché del provider. Una línea base puede publicar su diagnóstico de presupuesto con una lista de cambios vacía. Un lote dinámico sin ningún cambio confirmado no se inyecta en absoluto, y un toque posterior lo reintenta.

El evento de línea base inicial en sí no se reescribe. Sus cambios tipados siguen siendo autoritativos solo mientras ese evento esté en la superficie de sesión visible. Cuando la compactación ensombrece el evento, el siguiente pre-step entrante compone la línea base actual y la registra en la misma petición; un toque exitoso del sistema de archivos puede en su lugar volver a añadir un ámbito de línea base sin cambios o añadir su reemplazo o eliminación. El marcador de ámbito en memoria y la caché de versiones del provider solo seleccionan y aceleran las sondas. En el primer pre-step tras reanudar o remontar en caliente, se conserva una línea base visible compatible y se compara con los archivos retenidos por el renderizado completo actual. Los archivos sin cambios y omitidos por presupuesto no añaden nada; las adiciones, ediciones, eliminaciones fuera de línea y los archivos que salen del conjunto de presupuesto retenido añaden transiciones `set`, `replace` o `remove`. Una línea base visible incompatible es superada por una línea base actual completa, incluida una línea base vacía explícita cuando no queda ningún candidato. No hay watcher de archivos, por lo que un cambio en disco se hace visible en el siguiente toque exitoso de `read`, `write` o `edit`, cuando una sesión reanudada reconcilia su línea base, o cuando un pre-step entrante restaura una línea base ensombrecida.

## Configuración

```ts
export interface Config {
  dshHome?: string
  projectRootMarkers?: string[]
  maxBytes: number
  maxSourceBytes?: number
  instructionFileCandidates?: string[]
  localInstructionFileCandidates?: string[]
}
```

`maxBytes` es obligatorio para que cada despliegue haga su elección de presupuesto de prompt explícitamente. `maxSourceBytes` limita cada archivo de instrucciones fuente antes del renderizado y su valor por defecto es 1 MiB. `projectRootMarkers` por defecto es `['.git']`, e `instructionFileCandidates` por defecto es `['AGENTS.md', 'CLAUDE.md']`. En cada directorio de proyecto se carga todo candidato existente, y los candidatos cuyo contenido coincide con uno anterior tras recortar el espacio circundante se descartan, por lo que con los valores por defecto un `AGENTS.md` y un `CLAUDE.md` que comparten contenido se renderizan una vez (como `AGENTS.md`), mientras que hermanos genuinamente distintos se aplican ambos. `localInstructionFileCandidates` por defecto es `['AGENTS.local.md', 'CLAUDE.local.md']` y carga sus overlays existentes junto a los archivos base del mismo directorio (renderizados después de ellos) bajo la misma deduplicación por directorio; una lista vacía desactiva el overlay. Las entradas de candidato de ambas listas deben ser nombres de archivo del mismo directorio, por lo que las entradas vacías, `.`/`..` y las entradas que contienen `/` o `\\` se ignoran.

El archivo global de usuario es siempre `$DSH_HOME/AGENTS.md` sin overlay local; ambas listas de candidatos controlan solo los ámbitos de proyecto. `$DSH_HOME` por defecto es `~/.dsh`, y los prefijos configurados `~`, `~/...` y los de estilo Windows `~\\...` se expanden contra el directorio home del sistema operativo. Un presupuesto de renderizado no positivo o no finito desactiva tanto la carga de línea base como la dinámica; el `maxSourceBytes` configurado debe ser un entero positivo.

## Presupuesto y lecturas acotadas

El renderizado preserva primero los archivos de instrucciones más específicos. Descarta archivos más amplios completos antes de truncar el archivo más específico y emite un aviso visible `Workspace instruction budget ...` que nombra las rutas omitidas y truncadas. Los bytes renderizados nunca superan `maxBytes`.

El contenido de las instrucciones se lee a través de `streamText()` bajo `maxSourceBytes`, incluso cuando los metadatos del provider omiten el tamaño o un archivo crece después de su sonda de metadatos. Un archivo sobredimensionado se ignora; durante la reconciliación dinámica es temporalmente no disponible en lugar de eliminado. El plugin no mantiene caché de todo el proceso y nunca cachea la prosa de las instrucciones. Su caché de ámbito local a la sesión usa las versiones del provider solo como señal rápida de invalidación; tras la invalidación, el SHA-1 sobre la lectura acotada sigue siendo la identidad de contenido entre providers almacenada en la fuente estructurada del mensaje.

## Model Experience

### Contexto de línea base

#### Lo que ve el modelo

En la primera petición, el historial derivado contiene un mensaje duradero de rol de usuario con la cadena acotada de instrucciones globales de usuario y de proyecto en orden de amplio a específico. La reanudación reutiliza ese mensaje cuando su línea base visible es compatible.

##### Plantilla de instrucción de línea base

```markdown
<system-reminder>
The following workspace instructions may be relevant to your work. Use them as guidance when applicable. More specific instructions take precedence over broader ones. They do not override system, developer, or direct user instructions.

Instructions from: ~/.dsh/AGENTS.md

<user-global-instructions>

Instructions from: AGENTS.md

<project-instructions>
</system-reminder>
```

#### Efecto de tokens

La línea base renderizada se añade una vez y permanece en el historial derivado hasta la compactación. `maxBytes` acota el mensaje completo, los archivos más amplios se omiten antes de truncar el archivo más específico, y una cadena vacía contribuye cero tokens.

#### Efecto de caché KV

Solo añadidura tras el prefijo reutilizable existente. La reanudación preserva la reutilización cuando la identidad de la línea base visible es compatible; una identidad incompatible añade un reemplazo completo, por lo que los cambios de descubrimiento, precedencia, raíz de proyecto o presupuesto afectan a la reutilización solo desde esa posición del historial.

### Contexto de ámbito recién descubierto

#### Lo que ve el modelo

Tras una llamada exitosa de primera parte al sistema de archivos que alcanza un directorio más profundo, la siguiente petición incluye un `user/message` retenido con fuente que contiene el archivo de instrucciones recién aplicable.

##### Plantilla de instrucción adicional

```markdown
<system-reminder>
Additional instructions from: packages/app/AGENTS.md

These instructions apply to work under `packages/app`. Use them as guidance when relevant; more specific instructions take precedence. They do not override system, developer, or direct user instructions.

<nested-instructions>
</system-reminder>
```

#### Efecto de tokens

Cada ámbito descubierto añade tokens de historial acotados hasta la compactación. El contenido sin cambios se suprime mediante el estado de sesión visible más la comparación de versión/digest, y Code Mode difiere el mismo mensaje hasta después del resultado `run_code` externo y de su paso duradero envolvente.

#### Efecto de caché KV

Solo añadidura; el contenido recién visible sigue el prefijo de petición reutilizable y no invalida las entradas de caché KV existentes.

### Contexto de instrucciones cambiadas o eliminadas

#### Lo que ve el modelo

Un archivo cambiado produce `Updated instructions from: <path>` más su contenido de reemplazo. Un candidato que desaparece o se convierte en duplicado por directorio de un candidato anterior produce el aviso de eliminación siguiente.

##### Aviso de eliminación

```markdown
<system-reminder>
Instructions removed: packages/app/AGENTS.md

The previously loaded instructions from this file no longer apply.
</system-reminder>
```

#### Efecto de tokens

Cada cambio o eliminación confirmado es un mensaje de historial retenido acotado por `maxBytes`. Los fallos de provider no añaden ningún mensaje, y una actualización omitida por el presupuesto sigue siendo elegible para un toque posterior del sistema de archivos.

#### Efecto de caché KV

Solo añadidura; el contenido recién visible sigue el prefijo de petición reutilizable y no invalida las entradas de caché KV existentes.

## Limitaciones conocidas y trabajo diferido

- **El descubrimiento sigue herramientas fs estructuradas, no la navegación de shell** — un comando `bash` que cambia de directorio no dispara el descubrimiento de instrucciones anidadas porque la sintaxis de shell y el estado de shell por llamada no son un seam fiable del sistema de archivos.
- **La actualización se impulsa por toques** — no hay watcher; las ediciones externas se hacen visibles en el siguiente `read`, `write` o `edit` exitoso de primera parte, cuando la reanudación reconcilia una línea base visible, o cuando un pre-step entrante restaura una línea base ensombrecida.
- **La semántica de candidatos se mantiene deliberadamente pequeña** — los nombres en minúsculas, `.claude/rules/` y las importaciones `@path` no se interpretan; los ámbitos de proyecto cargan por defecto los overlays `AGENTS.local.md`/`CLAUDE.local.md`, pero el ámbito global de usuario `$DSH_HOME` no tiene overlay local y otros nombres personalizados requieren configuración explícita de candidatos.
- **La deduplicación por directorio se basa en el contenido** — los candidatos hermanos se colapsan solo cuando son byte-idénticos tras recortar el espacio en blanco inicial y final; un `CLAUDE.md` que hace symlink de su hermano `AGENTS.md` se resuelve al mismo contenido y se colapsa como cualquier duplicado, mientras que una copia real distinta que se ha desviado de `AGENTS.md` se carga completa junto a él.
- **Los archivos de instrucciones con symlink se siguen a través del límite de confianza** — un candidato cuyo componente final es un symlink se resuelve y su destino se carga, por lo que un repositorio clonado puede exponer contenido de archivos fuera del árbol como guía de workspace de menor autoridad (nunca anula las instrucciones de sistema, de desarrollador o directas del usuario). Confina `ctx.fs` con la puerta de política del sistema de archivos o un sandbox del SO al cargar repositorios no confiables.
- **El contenido de las instrucciones se acota, no se resume** — los archivos amplios que superan el presupuesto se omiten y el archivo más específico puede truncarse; el plugin nunca pide a un modelo que comprima la prosa de las instrucciones.
