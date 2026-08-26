# Agent Note: Convierte `dsh-fs-observation-policy` en un plugin de puerta de eventos, no en una interfaz de métodos

Status: implemented

[English](2026-06-26-file-context-as-event-gate.md) | Español

## Problema

El [Agent Note de división del seam de fs](../simplification/2026-06-26-fsspec-style-fs-seam.es.md) colocó `ctx.fileContext` entre las herramientas orientadas al modelo y el provider `ctx.fs`: `dsh-tool-fs` inyecta `fileContext` y enruta cada `read`/`write`/`edit` a través de sus métodos. Eso hace que `fileContext` esté **en la ruta y sea obligatorio**. La herramienta no puede llegar a `ctx.fs` sin él, la capa de política es dueña del I/O de fs y de la ventana de lectura, y un despliegue que no quiera la política de estado observado no puede simplemente descartar el paquete — `dsh-tool-fs` fallaría al resolver `ctx.fileContext`.

Esto acopla tres cosas que deberían poder separarse:

1. **Lo que hace la herramienta** — resolver una ruta, leer una ventana, escribir/editar un archivo. Es el trabajo de la herramienta y solo necesita `ctx.fs`.
2. **La política de frescura/observación** — «editar requiere una lectura previa», «escribir/editar debe basarse en la versión que leíste». Es el trabajo del plugin `dsh-fs-observation-policy`.
3. **El registro del estado observado** — un efecto secundario que nunca debería bloquear el funcionamiento de la herramienta.

Como la herramienta llama a los métodos de `fileContext`, eliminar la capa de política es un cambio disruptivo, no la pérdida elegante de un *añadido*. La política es estructural para que la herramienta siquiera se ejecute, no un endurecimiento opcional.

## Decisión

Invierte el flujo de control. **`dsh-tool-fs` pasa a ser el ejecutor y llama directamente a `ctx.fs`**; **`dsh-fs-observation-policy` pasa a ser un plugin de puerta + registro** que participa a través de eventos, nunca a través de un método que la herramienta llame ni registrando un servicio `ctx.fileContext`.

```text
tool          dsh-tool-fs       executor: resolves, reads windows, writes/edits via ctx.fs;
                                emits fs policy events; renders results
policy        dsh-fs-observation-policy  plugin: listens to fs/write-intent +
                                fs/edit-intent (single-slot waterfall) and fs/observed
                                (emit) events; adds observed-state + freshness.
provider contract dsh-fs            ctx.fs: text IO + ATOMIC mutation primitives whose version
                                guard is OPTIONAL; owns the fs policy event vocabulary
provider      dsh-fs-local      local implementation of ctx.fs
```

El modelo es aditivo: el `ctx.fs` pelado realiza I/O de texto atómico y sin restricciones, mientras que `dsh-fs-observation-policy` añade estado observado, lectura-antes-de-editar y protecciones de versión. Por tanto, eliminar la política deja las herramientas utilizables pero sin restricciones. Las configuraciones de agent distribuidas cargan la política; el modo pelado existe para mantener la política opcional en el límite del servicio, no como postura normal de despliegue.

El [seguimiento de observación de ausencia del sistema de archivos](../bug-fix/2026-08-09-filesystem-absence-observation.es.md) refina la carga útil de registro, de una versión solo-de-éxito a un estado explícito presente/ausente, y exige que la creación protegida publique sin reemplazo. La titularidad de la puerta de eventos y el límite de política sin I/O permanecen sin cambios.

`dsh-tool-fs` ya no inyecta `fileContext`. Inyecta `fs` y `tools`/`systemPrompt`.

## La política se aplica mediante el CAS del provider, no mediante el estado de `dsh-fs-observation-policy`

`dsh-fs-observation-policy` aplica «debes escribir/editar basándote en la versión que leíste» **sin llamar jamás a `stat` ni comparar versiones por su cuenta**. Aporta la versión observada como base del CAS y deja que la sección crítica de mutación del provider detecte la obsolescencia:

- «¿Qué observó por última vez este owner?» es lo único que `dsh-fs-observation-policy` decide localmente — una consulta a `WeakMap`, sin I/O. Sin registro significa no visto; un registro de ausencia solo permite la creación protegida; un registro de presencia lleva la base de reemplazo/edición.
- «¿Sigue siendo actual la versión, o el objetivo de creación sigue ausente?» se decide **dentro del límite de mutación atómica del provider**. `dsh-fs-observation-policy` aporta `replaceIfVersion` o `createIfAbsent`; el provider lanza `FS_STALE_VERSION` para una versión movida y `FS_NOT_OBSERVED` cuando una creación protegida pierde frente a otro creador.

Esto es deliberado. Si `dsh-fs-observation-policy` hiciera `stat` y comparara versiones en su handler de waterfall, habría una ventana TOCTOU entre esa comprobación y la escritura real de la herramienta — el archivo podría cambiar en medio, así que la comprobación sería una garantía falsa que el bloqueo del provider tendría que respaldar de todos modos. Poner la comprobación de versión en la sección crítica del provider es a la vez libre de carreras y con cero `stat` extra. Así que `dsh-fs-observation-policy` no hace **ningún** I/O del sistema de archivos; la garantía de «debe basarse en la última lectura» se *realiza* mediante CAS, y `dsh-fs-observation-policy` solo elige la base (`vObserved`) y aplica la puerta según la observación previa.

## Cambio del contrato del provider: la protección de versión es opcional

Para que el provider pelado no tenga restricciones, la protección de versión de sus dos mutaciones pasa a ser **opcional** — presente ⇒ protegida, ausente ⇒ incondicional:

```ts ignore-check
// writeText: expected is now optional. The FsWriteIntent union is UNCHANGED.
writeText(target: FsTarget, content: string, expected?: FsWriteIntent, signal?: AbortSignal): Promise<FsWriteOutcome>
//   undefined          → unconditionally create-or-overwrite (bare default)
//   createIfAbsent     → create only, reject an existing file (dsh-fs-observation-policy, unobserved)   [unchanged]
//   replaceIfVersion   → overwrite only at the observed version, else FS_STALE_VERSION    [unchanged]

// editText: expected becomes optional (was the required { version: FsVersion }).
editText(target: FsTarget, edit: FsEditRequest, expected?: { version: FsVersion }, signal?: AbortSignal): Promise<FsEditOutcome>
//   undefined    → unconditionally replace literal text in the current content (bare default);
//                  a missing target still reports FS_STALE_VERSION
//   { version }  → edit only at that version, else FS_STALE_VERSION (the current behavior)
```

La unión `FsWriteIntent` en sí no cambia — el tercer estado «incondicional» se expresa *omitiendo* `expected`, así que ambas mutaciones comparten una forma simétrica (`expected?`: omitido = sin protección, presente = protegido). Esto mantiene plena compatibilidad hacia atrás para las rutas protegidas que usa `dsh-fs-observation-policy`; solo el caso «sin protección», antes imposible, es nuevo, y es el valor por defecto del provider pelado. La mutación sigue ejecutándose dentro del bloqueo por objetivo del backend en cualquier caso, así que una escritura/edición incondicional sigue siendo atómica (sin archivos desgarrados); «incondicional» elimina la precondición de *versión*, no la atomicidad. `editText` informa un objetivo ausente como `FS_STALE_VERSION` tanto en la ruta protegida como en la no protegida, conservando un único código de fallo de edición para «el objetivo no se puede editar en este momento».

## Vocabulario de eventos (propiedad de `dsh-fs`)

Los eventos viven en `@deepseek-ai/dsh-fs`, no en `dsh-fs-observation-policy`. Lo impone el contrato de desacoplamiento: `dsh-tool-fs` es el emisor, así que debe referenciar los tipos de eventos, y debe seguir compilando aunque `dsh-fs-observation-policy` ya no proporcione un servicio de métodos. `dsh-fs` es el paquete del que ya dependen tanto `dsh-tool-fs` como `dsh-fs-observation-policy`, así que es el único hogar que permite al emisor y al listener de la política compartir un vocabulario sin que el emisor dependa del plugin de política.

Estos eventos transportan el vocabulario existente de `dsh-fs` (`FsTarget`, `FsVersion`, `FsObservation`, `FsWriteIntent`) más un actor opaco — no conceptos orientados al modelo (no se filtran ventanas de líneas, líneas numeradas ni pies renderizados).

**Los dos eventos de decisión `fs/*` son waterfalls de slot único en los que gana el primero.** `dsh-fs-observation-policy` retorna sin llamar a `next()`, así que es dueño del slot en el despliegue por defecto; un listener registrado antes o con `prepend` reemplazaría a esa política. Las preocupaciones de permiso, auditoría y sandbox permanecen en el waterfall componible `tools/execute`.

El actor está tipado como `object` en `dsh-fs` — un portador opaco puro que el contrato del provider nunca lee ni estrecha. La derivación de owner (`actor.agent?.session`) y la forma estructural `{ agent?: { session? } }` permanecen íntegramente dentro de `dsh-fs-observation-policy`, que estrecha el actor `object` a esa forma en sus listeners. `dsh-fs` es dueño de los nombres de eventos y del vocabulario de fs; NO es dueño de la estructura de owner en tiempo de ejecución de la capa de política.

```ts
import type { FsObservation, FsTarget, FsVersion, FsWriteIntent } from '@deepseek-ai/dsh-fs'

interface Events {
  /**
   * Single-slot decision: produce the write expectation for the next
   * ctx.fs.writeText. The default returns undefined (unconditional create-or-
   * overwrite — the bare provider). The policy listener returns createIfAbsent
   * (unobserved) or { kind: 'replaceIfVersion', version: vObserved } (observed).
   * The listener does NOT call next(): one decision, not a composable chain. @mode waterfall
   */
  'fs/write-intent'(target: FsTarget, actor: object | undefined, next: () => FsWriteIntent | undefined | Promise<FsWriteIntent | undefined>): Promise<FsWriteIntent | undefined>
  /**
   * Single-slot decision: produce the optional version guard for the next
   * ctx.fs.editText. The default returns undefined (unconditional edit of the
   * current content — the bare provider; no stat). The policy listener returns
   * { version: vObserved }, or throws FS_NOT_OBSERVED if the actor is unset or
   * has not observed the target. Does NOT call next(): one decision. @mode waterfall
   */
  'fs/edit-intent'(target: FsTarget, actor: object | undefined, next: () => { version: FsVersion } | undefined | Promise<{ version: FsVersion } | undefined>): Promise<{ version: FsVersion } | undefined>
  /**
   * Record that an actor observed a target as present at a version or absent.
   * Fire-and-forget (plain emit). Listeners MUST be
   * synchronous, side-effect-only recorders (`dsh-fs-observation-policy`'s is a WeakMap
   * write); the tool does not guard the emit, so a throwing listener surfaces as
   * the tool's isError result. No listener ⇒ nothing recorded.
   * @mode emit
   */
  'fs/observed'(target: FsTarget, observation: FsObservation, actor: object | undefined): void
}
```

Los eventos de decisión `fs/*` son **waterfalls sin enlazar que despacha la herramienta** (como `agent/request`, que el loop despacha sin `this`), no waterfalls enlazados a servicios (como `llm/stream`). El despachador es el plugin `dsh-tool-fs`, que no es un servicio.

## Contrato de la herramienta (`dsh-tool-fs`)

La herramienta conserva sus schemas orientados al modelo (`read`/`write`/`edit`, sin cambios byte a byte) y sus secciones de prompt. La guía del prompt sigue siendo de política primero porque se espera que un despliegue que carga las herramientas de fs cargue también `dsh-fs-observation-policy`: al modelo se le sigue indicando que lea antes de sobrescribir o editar, y ese requisito es del plugin fs-observation-policy, no del backend. El respaldo del provider pelado no cambia la postura del prompt.

`dsh-tool-fs` asume las responsabilidades de ejecutor trasladadas desde el antiguo servicio de métodos `fileContext`, incluido el **renderizado de lectura** (`read-render.ts`: `buildWindow` + `formatReadOutput`, `READ_MAX_BYTES`, `READ_MAX_LINE_LENGTH`, `FileReadOutcome`/`FileTextLine`, más `STREAM_MIN_SIZE` en `read.ts`), que es el detalle de renderizado de la herramienta ahora que la herramienta es dueña de la lectura. Esos tipos y helpers de renderizado de lectura pasan a `dsh-tool-fs`; el plugin de política no debe seguir siendo una dependencia de tipos para la herramienta.

`dsh-tool-fs` es un único plugin raíz que registra las tres herramientas (`read`/`write`/`edit`), en espejo de `dsh-tool-bash`. Inyecta `fs` (más `tools`/`systemPrompt`), nunca `fileContext`. (La propuesta original exponía además cada herramienta como plugin de subruta `/read`/`/write`/`/edit` para despliegues centrados; se descartó en la implementación — ningún consumidor necesitaba un despliegue de herramienta única, y la publicación por subrutas obligaba a un manejo a medida de `tsdown`/`tsconfig`/`files`/workspace-constraint que ningún paquete hermano de herramientas lleva. Los helpers de registro por herramienta (`applyReadTool`/`applyWriteTool`/`applyEditTool`) siguen siendo módulos internos que compone el plugin raíz.)

El presupuesto de `stat` se minimiza dejando que el waterfall produzca la expectativa de forma perezosa — el valor por defecto pelado retorna `undefined` (sin protección) y nunca hace `stat`:

- **read** — un `stat`; un fallo de metadatos emite `{ kind: 'absent' }` antes de retornar `FS_NOT_FOUND`, mientras que un archivo pasa por `readText`/`streamText`, `buildWindow`, y luego emite `{ kind: 'present', version: info.version }`. El `stat` de confirmación posterior a la lectura del antiguo `fileContext.read` sigue descartado; un escritor que corra entre el `stat` de enrutado y la lectura puede, en el peor caso, hacer que una edición protegida posterior quede obsoleta espuriamente.
- **write** — `expectation = await ctx.waterfall('fs/write-intent', target, exec, () => undefined)`, luego `ctx.fs.writeText(target, content, expectation)`, luego emitir la versión de resultado presente. **Cero `stat` en la herramienta**, con o sin `dsh-fs-observation-policy`.
- **edit** — `expectation = await ctx.waterfall('fs/edit-intent', target, exec, () => undefined)`, luego `ctx.fs.editText(target, edit, expectation)`, luego emitir la versión de resultado presente. **Cero `stat` en la herramienta** en ambos casos: el valor por defecto pelado es `undefined` (edición incondicional), así que la herramienta nunca hace `stat` para fabricar una base. Si el objetivo está ausente en la ruta pelada, el provider informa `FS_STALE_VERSION`; la política retorna `FS_NOT_FOUND` directamente cuando ya tiene una observación de ausencia.

La herramienta pasa `exec` (el contexto de ejecución de la herramienta) como argumento `actor` en cada despacho, para que `dsh-fs-observation-policy` pueda derivar su owner del estado observado. La herramienta no sabe si el plugin de política está presente: siempre aporta el comportamiento por defecto pelado en el thunk `next`, y `dsh-fs-observation-policy` cortocircuita el thunk antes de que se ejecute en el despliegue por defecto.

**`fs/observed` se dispara tras una operación exitosa y tras confirmar la ausencia mediante una sonda de metadatos.** Sus listeners deben ser registradores síncronos que no lancen; la herramienta no protege la emisión simple, así que un listener que lanza puede reemplazar el error de lectura pendiente o informar un fallo después de que una mutación ya haya tenido éxito. La observación asíncrona o falible necesita un contrato de eventos aparte.

## Contrato del plugin de política (`dsh-fs-observation-policy`)

`dsh-fs-observation-policy` es un plugin, no un servicio. No registra `ctx.fileContext`, no tiene superficie pública de métodos y no expone métodos `read`/`write`/`edit`/`resolve`. Adjunta tres listeners mediante registros `ctx.on()` (cada uno retorna un disposer para HMR). Conserva el `WeakMap<owner, Map<targetKey, FsObservation>>` del estado observado y la derivación estructural de owner (estrechando el actor opaco `object` del evento a su propia forma `{ agent?: { session? } }`), pero no inyecta `fs` — cada handler opera solo sobre su propio `WeakMap`, nunca sobre `ctx.fs`.

- listener de `fs/write-intent`: no visto/ausente ⇒ `createIfAbsent`; presente ⇒ `replaceIfVersion`. NO llama a `next()`: es dueño absoluto del slot único de decisión.
- listener de `fs/edit-intent`: no visto ⇒ `FS_NOT_OBSERVED`; ausente ⇒ `FS_NOT_FOUND`; presente ⇒ su protección de versión. NO llama a `next()`.
- listener de `fs/observed`: registrar el valor discriminado presente/ausente.

Una entrada de estado observado es el **registro de observación previa**, pero su discriminante importa. Una lectura/escritura/edición exitosa registra presente en una versión, lo que permite crear-y-luego-editar o editar-y-luego-editar sin una lectura intermedia. Una lectura/vista que confirma la ausencia reemplaza cualquier versión positiva antigua por ausente, permitiendo solo una creación protegida; una creación exitosa posterior la reemplaza por la nueva versión presente. Una entrada ausente por sí sola significa no visto y produce `FS_NOT_OBSERVED` para la edición. El owner se deriva estructuralmente de `{ agent?: { session? } }`; la disposición descarta todo el estado (seguridad de HMR).

`dsh-fs-observation-policy` es ahora un plugin puro de política/registro sin API de servicio — solo influye en el mundo a través de la puerta de eventos. Eso es lo que elimina el acoplamiento de métodos de `dsh-tool-fs`.

## Comportamiento del provider pelado (sin `dsh-fs-observation-policy`)

Esta no es la postura de despliegue prevista — se espera que una configuración que carga las herramientas de fs cargue también `dsh-fs-observation-policy`. Es el suelo del provider sin restricciones que existe una vez que la herramienta ya no está acoplada a un servicio de métodos de política. Con `dsh-fs-observation-policy` ausente, todo waterfall `fs/*` cae en su valor por defecto `undefined` y `fs/observed` no tiene listener:

- **read** es idéntica (nunca necesitó política; solo emite un `fs/observed` ahora inaudito).
- **write** crea-o-sobrescribe incondicionalmente: `expected` es `undefined`, así que `writeText` escribe exista o no el archivo y sea cual sea su versión actual. Sin requisito de leer primero, sin comprobación de versión.
- **edit** reemplaza incondicionalmente el texto literal en el contenido actual del archivo: `expected` es `undefined`, así que `editText` empareja y reescribe sin protección de versión ni requisito de leer primero (`FS_EDIT_NOT_FOUND`/`FS_AMBIGUOUS_EDIT` siguen aplicando — se refieren al emparejamiento literal, no a la frescura). Un objetivo ausente sigue informando `FS_STALE_VERSION`, igualando el código «no se puede editar este objetivo ahora» de la ruta de edición protegida.

Ambas mutaciones siguen siendo atómicas (el bloqueo por objetivo del backend es incondicional). Lo que simplemente está *ausente*, no perdido, es la política que añadiría `dsh-fs-observation-policy`: estado observado, lectura-antes-de-editar y escritura/edición protegida por versión. Cargar `dsh-fs-observation-policy` superpone esas restricciones haciendo que sus listeners retornen valores `expected` protegidos en lugar de `undefined`; nada del provider pelado cambia.

## Sustituye

Esto enmienda — no revierte — [el Agent Note de división del seam de fs](../simplification/2026-06-26-fsspec-style-fs-seam.es.md). Se conservan la división en cuatro capas, el contrato del provider y la *política* de frescura. Lo que cambia es el **acoplamiento entre la herramienta y la capa de política**: un servicio de métodos obligatorio pasó a ser una puerta de eventos propiedad del plugin, y el I/O de fs + la ventana de lectura se trasladaron desde `fileContext` hacia `dsh-tool-fs`. La descripción del Agent Note de división del seam de fs de `dsh-tool-fs` inyectando `fileContext` y de `fileContext` siendo dueño de `read`/`write`/`edit` se actualizó para coincidir en el mismo cambio.

## Verificación

Los tests fijan ambas rutas: sin `dsh-fs-observation-policy`, el plugin raíz de herramientas arranca contra `dsh-fs-local`, y la lectura, la creación, la sobrescritura y la edición sin leer tienen éxito; con la política, la edición sin leer retorna `FS_NOT_OBSERVED` y la sobrescritura sin leer queda sujeta a `createIfAbsent`. Un listener de intención posterior no se alcanza después de que la política decida. Las ediciones obsoletas fallan mediante el CAS del provider mientras la política no realiza ningún `stat`; los presupuestos de la herramienta siguen siendo un `stat` para la lectura y cero para la escritura o la edición en cualquiera de las dos rutas. La ruta de recuperación de borrado también está ensamblada: mutación obsoleta, relectura ausente, recreación protegida. Los schemas orientados al modelo permanecen sin cambios byte a byte, mientras que el transcript de resultado recuperado cambia.

## Alternativas consideradas

- **Mantener `ctx.fileContext` como servicio de métodos en la ruta** — la forma con la que [el Agent Note de división del seam de fs](../simplification/2026-06-26-fsspec-style-fs-seam.es.md) aterrizó inicialmente; rechazada porque la herramienta no podía ejecutarse sin la capa de política, convirtiendo la política en estructural para la operación básica en lugar de un endurecimiento opcional.
- **Comprobación de versión en el lado de la política** (`dsh-fs-observation-policy` hace `stat` y compara en su handler de waterfall) — rechazada por la ventana TOCTOU entre esa comprobación y la escritura real de la herramienta; la sección crítica de mutación del provider es el único lugar libre de carreras, así que la política solo elige la base del CAS y aplica la puerta según la observación previa.
- **Plugins de subruta `/read`/`/write`/`/edit` por herramienta** — descartados en la implementación: ningún consumidor necesitaba un despliegue de herramienta única, y la publicación por subrutas obligaba a un manejo a medida de `tsdown`/`tsconfig`/`files`/workspace-constraint que ningún paquete hermano de herramientas lleva; los helpers de registro por herramienta siguen siendo módulos internos que compone el plugin raíz.

## Consecuencias

- **Indirección de eventos sobre una llamada a método.** Un waterfall + emit es menos directo que `await ctx.fileContext.edit(...)`. La recompensa es eliminar la dependencia de métodos entre herramienta y política manteniendo el plugin de política por defecto; el coste es un vocabulario de eventos más que aprender. Mitigado manteniendo los tres eventos acotados y documentando en cada uno la semántica del thunk por defecto.
- **Eventos de política en el seam de almacenamiento.** `dsh-fs` gana dos eventos de decisión de versión más un evento de registro aunque sea «solo almacenamiento». Es el precio del desacoplamiento (el emisor no puede depender del plugin de política). Los eventos transportan solo vocabulario de `dsh-fs` más un actor opaco `object` y ningún concepto orientado al modelo, así que el seam permanece libre de tipos de ventana de líneas/política de observación y de la estructura de owner agent/session.
- **Un solo ocupante de política, gana el primero por convención.** Los slots `fs/write-intent`/`fs/edit-intent` albergan exactamente un decisor; gana el listener registrado primero (o con `prepend`) y el resto quedan cortocircuitados. Que `dsh-fs-observation-policy` sea dueño del slot es una convención de despliegue, no un invariante impuesto por eventos — un segundo decisor registrado primero lo sortearía. Esto es aceptable porque un segundo decisor de política de versión de fs es una mala configuración, no una funcionalidad. Si en el futuro aparece una necesidad de política de versión de fs *en capas*, será un Agent Note nuevo (un waterfall componible que pasa valores), no un segundo listener silencioso en estos eventos. La intercepción en capas de permiso/auditoría/sandbox ya tiene su hogar en `tools/execute`.
- **Descartar el `stat` de confirmación posterior a la lectura** hace que una edición *protegida* posterior falle a veces en modo cerrado (`FS_STALE_VERSION` → relectura) bajo una carrera de lectura/escritura. Es un refinamiento de UX perdido, nunca un agujero de corrección; el bloqueo del provider sigue impidiendo escrituras con versión incorrecta.
- **El provider pelado no hace lectura-antes-de-escribir/editar ni comprobación de versión.** Un despliegue sin `dsh-fs-observation-policy` deja que el modelo sobrescriba o edite incondicionalmente cualquier archivo existente. Este es el significado deliberado de mantener la herramienta independiente de un servicio de política: las disciplinas de seguridad viven en el plugin `dsh-fs-observation-policy`. Un despliegue que lo omite está optando a propósito por un sistema de archivos sin restricciones; no es la postura prevista para una configuración que distribuye las herramientas de fs.
