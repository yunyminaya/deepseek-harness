# Agent Note: Seam de capacidad de filesystem — ctx.fs, backend local y herramientas de filesystem orientadas al modelo

Status: implemented

[English](2026-06-17-filesystem-capability-seam.md) | Español

## Problema

El harness tiene un seam de capacidad `bash` concreto (`dsh-shell` / `dsh-bash-local` / `dsh-tool-bash`), pero las operaciones de filesystem estaban a punto de aterrizar como herramientas orientadas al modelo sin un seam equivalente. Si `read`, `write` y `edit` usaran directamente `node:fs`, el paquete de herramientas orientado al modelo sería dueño a la vez de la política de ejecución del filesystem, la resolución de rutas locales, el comportamiento de escritura atómica, la decodificación de texto, el comportamiento de symlinks y la semántica de edición.

Eso acopla tres preocupaciones que cambian de forma independiente:

1. El contrato del filesystem: qué operaciones pueden pedir los plugins.
2. El backend: disco local ahora, filesystem en sandbox/remoto/acotado a proyecto después.
3. La API del consumidor: los schemas `read` / `write` / `edit` orientados al modelo y el formateo de resultados.

Sin una interfaz `ctx.fs`, sustituir el acceso local al filesystem por un backend en sandbox o remoto obligaría a remover los schemas de herramienta, las demos y las guías de prompt incluso cuando el contrato orientado al modelo debería permanecer estable. También hace más difícil razonar sobre los límites de permiso/sandbox: una opción `cwd` puede parecer un sandbox aunque solo sea una ruta base, a menos que un backend explícito o una política de `tools/execute` impongan el confinamiento.

Las herramientas de filesystem deben aterrizar con la misma forma de seam de capacidad que bash antes de convertirse en una superficie pública de paquetes.

## Decisión

El acceso al filesystem es un seam de capacidad de primera clase que sigue [el Agent Note de seams de capacidad](2026-06-13-capability-seams.es.md):

1. `@deepseek-ai/dsh-fs` (`packages/fs/fs`) es dueño del servicio abstracto `ctx.fs`, de los tipos de vocabulario del filesystem y del vocabulario de eventos de política `fs/*`.
2. `@deepseek-ai/dsh-fs-local` (`packages/fs/fs-local`) proporciona la primera implementación, respaldada por el filesystem local.
3. `@deepseek-ai/dsh-tool-fs` (`packages/fs/tool-fs`) proporciona las herramientas `read`, `write` y `edit` orientadas al modelo sobre `ctx.fs`, y es el ejecutor que despacha los eventos `fs/*`.

El paquete Consumer depende solo del paquete Service Definition, nunca de `dsh-fs-local`. Un despliegue que quiera otro backend carga un provider distinto para `ctx.fs` sin cambiar los schemas de herramienta ni las guías de prompt orientadas al modelo.

La política de leer-antes-de-escribir/editar y de estado observado es un cuarto paquete, `@deepseek-ai/dsh-fs-observation-policy` (`packages/fs/fs-observation-policy`), aportado a través de la puerta de eventos `fs/*` en lugar de vivir en `ctx.fs`; un despliegue que carga `dsh-tool-fs` carga también `dsh-fs-observation-policy` para obtener leer-antes-de-escribir/editar. Esta decisión fijó el límite de tres paquetes; la separación de la política fuera de la clase base del provider la decide [el Agent Note de separación del seam fs](../simplification/2026-06-26-fsspec-style-fs-seam.es.md), y su realización como plugin de puerta de eventos (no como servicio de métodos), [el Agent Note de puerta de eventos](2026-06-26-file-context-as-event-gate.es.md).

El primer backend es deliberadamente solo local: `dsh-fs-local` implementa `ctx.fs` contra el filesystem del host. Futuros backends hermanos pueden proporcionar filesystems en sandbox, remotos, virtuales o acotados a proyecto detrás de la misma interfaz.

El primer Consumer es deliberadamente solo de archivos de texto: `dsh-tool-fs` expone las herramientas `read`, `write` y `edit` orientadas al modelo para archivos de texto UTF-8. Los Consumers futuros pueden añadir listado de directorios, búsqueda/glob, operaciones seguras con binarios, vigilancia de archivos u operaciones de proyecto de nivel superior sin cambiar el paquete del backend local, siempre que la capacidad necesaria exista en `ctx.fs`. El listado directo de directorios se añadió después mediante [Añadir listado directo de directorios al seam de filesystem](../../archived/architecture/2026-07-03-filesystem-directory-listing-seam.md).

Los permisos y el sandboxing del filesystem no están implícitos en esta separación. El backend local resuelve las rutas relativas desde su directorio base configurado, pero la política de confinamiento es una decisión aparte: o una implementación de `ctx.fs` más estricta la impone, o un plugin de permiso/sandbox envuelve `tools/execute` y veta las llamadas antes de que lleguen al consumidor.

Leer-antes-de-escribir/editar y el estado observado pertenecen a `dsh-fs-observation-policy`, no a `ctx.fs`. A través de la puerta de eventos `fs/*`, la política registra versiones por actor opaco y suministra expectativas de mutación opcionales; el provider impone la frescura de forma atómica. `dsh-tool-fs` emite los eventos sin depender de la política. Consulta los Agent Notes de [separación del seam](../simplification/2026-06-26-fsspec-style-fs-seam.es.md) y [puerta de eventos](2026-06-26-file-context-as-event-gate.es.md).

## Topología de paquetes

El seam de filesystem usa la misma dirección de dependencias que el trío de bash:

```text
@deepseek-ai/dsh-tool-fs  --depends on-->  @deepseek-ai/dsh-fs  <--depends on--  @deepseek-ai/dsh-fs-local
        consumer                                interface                         implementation
```

`@deepseek-ai/dsh-fs` depende solo de `cordis` más la base `HarnessError` de todo el repo de `@deepseek-ai/dsh-llm`. Declara la clave `ctx.fs`, el servicio abstracto `FileSystem`, los tipos de vocabulario compartidos por backends y consumers, el vocabulario de errores del filesystem y el vocabulario de eventos de política `fs/*`. No lleva ningún almacén de estado observado ni ninguna forma de derivación de dueño; los eventos pasan un actor `object` opaco que el provider nunca lee, y el plugin `dsh-fs-observation-policy` es dueño de la forma de derivación de dueño y del almacén de estado observado sobre esos eventos.

`@deepseek-ai/dsh-fs-local` depende de `@deepseek-ai/dsh-fs` y de `cordis`. Subclasea `FileSystem`, se registra a sí mismo como `ctx.fs`, es dueño de la configuración del backend local, como el directorio base, y contiene todo el acceso directo a `node:fs` / `node:path`. No guarda ningún almacén de estado observado — la frescura es un token de versión que el backend acuña y que el plugin de política registra.

`@deepseek-ai/dsh-tool-fs` depende de `@deepseek-ai/dsh-fs`, `@deepseek-ai/dsh-tools`, `@deepseek-ai/dsh-system-prompt` y `cordis`. Registra las herramientas orientadas al modelo y las secciones de prompt. No debe importar `node:fs`, `node:path` ni `@deepseek-ai/dsh-fs-local`; la ejecución del filesystem siempre pasa por `ctx.fs`. Si la implementación necesita tipos concretos de helper de agent o de sesión, esas dependencias pertenecen a `tool-fs`; no deben filtrarse de vuelta a `dsh-fs`.

El plugin raíz `tool-fs` registra la suite completa de herramientas de filesystem (`read`, `write` y `edit`) componiendo los helpers de registro por herramienta. Inyecta `fs` y nunca importa un paquete de Service Provider.

## Contrato de `ctx.fs`

`@deepseek-ai/dsh-fs` es dueño de un servicio de filesystem semántico. Es de nivel más alto que `readFile` / `writeFile`, de modo que `tool-fs` no reimplementa la resolución de rutas, el versionado, la decodificación de texto, el rechazo de binarios, la paginación, el reemplazo atómico, el comportamiento de symlinks ni la semántica de edición literal.

La interfaz cubre estas operaciones semánticas:

- Resolver una ruta suministrada por el modelo/plugin a un target definido por el backend.
- Convertir un target resuelto a la ruta canónica del proceso o al URI `file:` para el mismo mundo de ejecución, y probar el confinamiento sin parsear su clave opaca.
- Obtener con stat la metadata del target sin leer el contenido del archivo.
- Leer texto UTF-8 completo o en streaming; los consumers aplican sus propios límites de vista y retención.
- Crear o reemplazar un archivo de texto UTF-8.
- Editar un archivo de texto UTF-8 existente mediante reemplazo literal.

El contrato del provider también lleva los hooks de frescura sobre los que se construye la política — pero el almacén de estado observado y la derivación del dueño viven en el plugin `dsh-fs-observation-policy`, no en `ctx.fs`:

- El backend acuña un token `version` opaco por target (en `stat` y en cada resultado de lectura/mutación).
- `writeText`/`editText` aceptan una expectativa de versión OPCIONAL: omítela para una mutación incondicional de provider puro, o suminístrala para proteger la mutación dentro de la sección crítica atómica del backend.
- El plugin `dsh-fs-observation-policy` decide esa expectativa en `fs/write-intent`/`fs/edit-intent` y registra las versiones observadas en `fs/observed`, indexadas por un dueño que deriva del actor opaco del evento (normalmente `exec.agent.session`).

La autorización es frescura de versión, no una distinción de vista completa/parcial: cualquier lectura registra la versión del target, y una escritura/edición posterior está autorizada mientras el archivo siga en esa versión — así, una lectura por ventana de las líneas 100-150 autoriza una edición de la línea 120. El almacén de estado observado es un `WeakMap<owner, Map<targetKey, version>>` dentro de `dsh-fs-observation-policy`; `dsh-fs` no guarda nada de eso y trata al actor como opaco. (Esta decisión modeló primero una caché `FileState` con vistas `full`/`partial` en `ctx.fs`; los notes de split-fs-seam y event-gate la sustituyeron por el plugin de política basado en frescura que se describe aquí.)

La resolución de rutas es explícita y puede ser asíncrona. La resolución local puede limitarse a normalizar una ruta, pero los backends en sandbox/remotos/acotados a proyecto pueden necesitar I/O para resolver una ruta suministrada por el usuario en una identidad de target estable.

Los targets resueltos deben exponer al menos tres conceptos:

- La ruta de entrada original, para diagnósticos.
- Un `targetKey` opaco, usado para las guardas de obsolescencia y la consulta de estado del archivo. El backend local podría usar una clave tipo realpath; un backend remoto podría usar un URI de workspace o un file id. Los consumers no deben parsearlo ni asumir que es una ruta absoluta local.
- Un `displayPath`, usado para la salida orientada al modelo/UI. Puede ser una ruta absoluta local, una ruta relativa al workspace o un URI remoto según el backend.

`targetKey` permanece opaco incluso cuando otra capacidad comparte el mundo de ejecución del provider. Esos consumers le piden al provider `processPath(target)`, `fileUrl(target)` o `contains(parent, child)`; la [decisión del mundo de ejecución portable](2026-07-28-portable-execution-world-consumers.es.md) es dueña de por qué estos hechos residen en el seam de filesystem.

Los resultados de lectura y mutación deben incluir una `version` de archivo opaca. El backend local deriva su token de la metadata bigint de stat (`dev`, `ino`, `size`, `mtimeNs` y `ctimeNs`), de modo que las reescrituras del mismo tamaño y el reemplazo de inode invalidan a los consumers de forma fiable; un backend remoto puede usar un revision id o un token tipo hash. El plugin `dsh-fs-observation-policy` registra las versiones para las comprobaciones de obsolescencia; los consumers pueden mostrar metadata relacionada, pero no deben interpretar el token de versión.

El provider devuelve texto decodificado: `readText` devuelve un archivo de texto regular completo y `streamText` transmite las mismas semánticas de texto para archivos grandes o límites de retención propiedad del consumidor. Las ventanas de líneas, los topes de bytes, el renderizado con numeración de líneas y el cómputo total de líneas viven en consumers como `dsh-tool-fs` y `dsh-lsp-stdio`. El provider es dueño de las comprobaciones de archivo regular, la decodificación UTF-8 y el rechazo de binarios/NUL; no sabe nada de ventanas de líneas, límites de protocolo ni vistas.

El registro del estado observado no está en `ctx.fs`: después de una lectura satisfactoria el ejecutor emite `fs/observed`, y el plugin `dsh-fs-observation-policy` registra `{ version }` para el dueño derivado. No existe vista `full`/`partial` — una lectura en cualquier ventana registra la versión, y la frescura (no la completitud de la vista) autoriza una escritura/edición posterior.

Las escrituras de archivo completo crean o reemplazan archivos de texto UTF-8. Los backends pueden crear los directorios padre cuando ese comportamiento esté soportado y documentado. Los targets existentes no regulares se rechazan. `writeText` acepta una expectativa opcional: `createIfAbsent` crea un target ausente y rechaza uno existente con `FS_NOT_OBSERVED` (la ruta que usa la política para un dueño no observado); `replaceIfVersion` reemplaza solo cuando el target existe en la versión observada; si no, `FS_STALE_VERSION`; omitir la expectativa es la creación-o-sobrescritura incondicional de provider puro. El plugin de política elige qué expectativa suministrar a partir del estado observado del dueño.

La edición literal es una primitiva del provider (`editText`), no se compone en `tool-fs` a partir de una lectura más una escritura. El emparejamiento literal, el rechazo de coincidencias duplicadas, la preservación de CRLF, el rechazo de binarios, la comprobación opcional de versión obsoleta y la lectura-modificación-escritura atómica deben permanecer juntos dentro de la sección crítica de mutación del backend. `editText` acepta la misma expectativa de versión opcional; la comprobación de obsolescencia se ejecuta antes del emparejamiento literal, de modo que una edición contra una lectura antigua informa `FS_STALE_VERSION`. Un backend remoto puede implementar la edición como una operación nativa de comparar-y-editar; el consumidor no fuerza la composición estilo local.

El plugin de política, no `ctx.fs`, condiciona la observación previa: un `edit` exige una observación previa del dueño (si no, `FS_NOT_OBSERVED`), y la versión registrada se pasa a `editText` como base CAS. Sin el plugin de política, `ctx.fs` por sí solo es un seam completo sin restricciones (escritura/edición incondicionales); la herramienta nunca queda acoplada por métodos a la política.

Los fallos del contrato del filesystem se lanzan como `FsError extends HarnessError`, y el registro de herramientas los convierte en resultados de herramienta `isError` con metadata estructurada `{ name, code }`. `dsh-fs` es dueño de este vocabulario en lugar de que cada herramienta invente mensajes. Los códigos son `FS_NOT_FOUND`, `FS_NOT_TEXT`, `FS_STALE_VERSION`, `FS_NOT_OBSERVED`, `FS_NOT_REGULAR_FILE`, `FS_AMBIGUOUS_EDIT`, `FS_EDIT_NOT_FOUND` y `FS_ABORTED`. (Un borrador anterior incluía `FS_PARTIAL_OBSERVATION`; la autorización basada en frescura no tiene distinción parcial/completa, así que se eliminó. Los códigos específicos de listado de directorios se añadieron después mediante [Añadir listado directo de directorios al seam de filesystem](../../archived/architecture/2026-07-03-filesystem-directory-listing-seam.md).)

## Comportamiento del consumidor de herramientas

`@deepseek-ai/dsh-tool-fs` es el consumidor orientado al modelo. Es dueño de los nombres de herramientas, los JSON schemas, la validación de argumentos en el límite del modelo, las secciones de prompt y el formateo de resultados. No es dueño de la ejecución del filesystem.

La primera suite de herramientas contiene:

- `read`: inspeccionar un archivo de texto UTF-8 y devolver el contenido con números de línea y orientación de paginación.
- `write`: crear o reemplazar por completo un archivo de texto UTF-8.
- `edit`: actualizar un archivo de texto UTF-8 existente reemplazando texto literal, exigiendo una coincidencia única por defecto y permitiendo un modo explícito de reemplazar-todo.

Cada herramienta sigue la misma forma de ejecución:

1. Validar y normalizar los argumentos del modelo.
2. Llamar a la operación adecuada de `ctx.fs`.
3. Formatear el resultado como `ContentBlock[]` para el modelo.
4. Dejar que los errores lanzados de backend/herramienta fluyan por `ToolRuntime.execute()`, que los convierte en resultados de herramienta `isError`.

El paquete registra la guía de prompt mediante `ctx.systemPrompt.section(...)` y registra los schemas mediante `ctx.tools.register(...)`. Los schemas de herramientas siguen fluyendo hacia la ruta normal de ensamblado del prompt vía `SystemPrompt.assemble()` y `ToolRuntime.schemas()`; no se requieren cambios en el agent loop.

El paquete de herramientas mantiene estables los contratos orientados al modelo cuando cambian los backends: un backend local y uno remoto pueden resolver las rutas de forma distinta internamente, pero los schemas `read` / `write` / `edit` no cambian solo porque cambie el backend.

El despliegue por defecto exige un `read` previo antes de actualizar un archivo existente con `write` o `edit`. `tool-fs` no lo implementa comprobando si se ejecutó una herramienta llamada `read`: despacha los eventos `fs/write-intent`/`fs/edit-intent` (pasando el contexto de ejecución como actor opaco), y el plugin `dsh-fs-observation-policy` deriva el dueño, condiciona la observación previa y suministra la expectativa de versión. Cualquier lectura por ventana autoriza una escritura/edición posterior mientras el archivo no cambie. Crear un archivo nuevo con `write` no exige observación previa.

El plugin raíz registra la suite completa componiendo los helpers de registro por herramienta. Inyecta `fs`, `tools` y `systemPrompt`.

## Pruebas

Las pruebas siguen el límite de paquetes, no solo las herramientas visibles al usuario: el contrato de servicio en `dsh-fs`; el comportamiento real del filesystem a través de la interfaz `ctx.fs` en `dsh-fs-local` (resolución, symlinks, streaming, rechazo de binarios/UTF-8, escrituras incondicionales y protegidas por versión, semántica de edición literal, preservación de finales de línea, códigos estructurados de `FsError`); la superficie del consumidor en `dsh-tool-fs` contra el provider local real (haz mock solo del modelo/reloj, nunca del colaborador); e integración mediante `ctx.tools.execute()` con y sin `dsh-fs-observation-policy`, verificada sobre el mundo real leyendo los archivos de vuelta desde el disco en lugar de confiar en el valor canónico o en el contenido renderizado. La política de estado observado/derivación del dueño se prueba en `dsh-fs-observation-policy`, no aquí.

Las clases de patrones defensivos por las que este repo ha sido mordido quedan fijadas directamente:

- **Seguridad del archivo temporal en escritura atómica.** Las escrituras/ediciones se intermedian con un directorio privado aleatorio `0700` junto al target, con un archivo temporal exclusivo solo del propietario (`'wx'`, `0o600`), limpieza en caso de fallo y un rename atómico final — reflejando las reglas de spill files de bash, porque las rutas temporales predecibles y legibles por todos invitan a carreras de symlinks y divulgación. Las pruebas afirman los permisos y que una ruta temporal preexistente no se pisa; esta primitiva es un requisito permanente del seam.
- **Identidad de `targetKey` a través de symlinks.** Dos rutas de entrada que resuelven al mismo realpath comparten una entrada de estado observado: un `read` por la ruta A satisface la guarda de leer-antes-de-editar para un `edit` por la ruta B del symlink, y una escritura obsoleta por una ruta se detecta por la otra.
- **Concurrencia / carreras de obsolescencia.** Dos operaciones concurrentes de escritura/edición contra el mismo target se resuelven de forma determinista — una tiene éxito y la otra se rechaza con `FS_STALE_VERSION` — y una edición satisfactoria refresca el estado registrado, de modo que la siguiente edición del mismo dueño procede.
- **Seguridad HMR y liberación de recursos.** Liberar el fiber del backend retira el provider `ctx.fs`; un provider posterior arranca sin estado heredado.

## Alternativas consideradas

- **Herramientas orientadas al modelo directamente sobre `node:fs`** — el paquete de herramientas sería dueño a la vez de la política de ejecución, la resolución de rutas, las escrituras atómicas, la decodificación de texto y la semántica de edición, acoplando las tres preocupaciones que cambian de forma independiente que nombra el Problema y removiendo los schemas en cualquier cambio de backend.
- **Un único paquete combinado `dsh-fs-tools`** — la forma previa al seam; rechazado por la misma separación de Service Definition / Service Provider / Consumer que bash, y el nombre combinado nunca llegó a ser API pública.
- **Estado observado en `ctx.fs`** — la forma con la que este Agent Note aterrizó primero; superada por [el Agent Note de separación del seam fs](../simplification/2026-06-26-fsspec-style-fs-seam.es.md) y [el Agent Note de puerta de eventos](2026-06-26-file-context-as-event-gate.es.md): un backend en sandbox/remoto no debe heredar la política de observación orientada al modelo, así que el provider conserva solo el token de versión y la mutación opcional protegida por versión.

## Consecuencias

**`cwd` puede confundirse con un sandbox.** El directorio base del backend local es un valor por defecto de resolución, no automáticamente un límite de confinamiento. Si se exige confinamiento, debe imponerlo el contrato del backend o un plugin de permiso/sandbox en `tools/execute`.

**La interfaz puede volverse demasiado local.** Devolver campos como `absolutePath` desde `ctx.fs` haría incómodos a los backends remotos, en sandbox o virtuales. El contrato debería exponer metadata de visualización sin exigir que los consumers entiendan rutas del host.

**La interfaz puede volverse demasiado delgada.** Si `ctx.fs` solo refleja las primitivas de `node:fs`, `tool-fs` reimplementará la detección de binarios, la paginación, las escrituras atómicas y la semántica de edición. Eso recrea el acoplamiento que esta decisión evita.

**La semántica de edición es propensa a carreras por naturaleza.** La edición literal es una operación de lectura-modificación-escritura; la guarda es la sección crítica de mutación atómica del backend más la expectativa de versión opcional, así que las ediciones concurrentes se resuelven de forma determinista — una gana y la otra recibe `FS_STALE_VERSION`.

**El estado observado no pertenece a `ctx.fs`.** Registrar lo que un contexto de ejecución ha visto es política de flujo de trabajo, no I/O crudo del filesystem. Esta decisión lo colocó primero dentro del seam de filesystem; el note de separación del seam estableció después que un backend en sandbox/remoto no debe heredar la política de observación orientada al modelo, y lo trasladó al plugin `dsh-fs-observation-policy`. El contrato del provider conserva solo lo que la seguridad de escritura/edición necesita de verdad en la capa de almacenamiento — un token de versión acuñado por el backend y una mutación opcional protegida por versión — mientras que el plugin de política es dueño de la derivación del dueño, el estado observado y la condición de leer-antes-de-editar sobre los eventos `fs/*`.

**La forma de resolver-y-luego-operar cuesta un round-trip extra por llamada.** Cada herramienta puede resolver una ruta a un `FsTarget` y luego emitir la lectura/escritura/edición como una llamada separada a `ctx.fs`. Para el backend local esto es despreciable (la resolución es normalización de rutas en memoria), pero un backend remoto/en sandbox puede convertir cada paso en su propia petición, así que un único `read` puede convertirse en dos round-trips de red. Los backends donde el round-trip importa pueden cachear o plegar la resolución internamente preservando el contrato observable.

**La persistencia del estado observado queda diferida.** El estado observado vive en memoria (el `WeakMap` dentro de `dsh-fs-observation-policy`), así que una sesión reanudada exige de forma conservadora volver a leer los archivos antes de escribir/editar, hasta que un futuro evento de sesión o mecanismo de persistencia haga reproducible la observación.

**Los códigos de error pasan a formar parte del seam.** Los códigos de `FsError` hacen que los fallos de versión obsoleta y de observación sean encaminables por máquina a través de la taxonomía de errores estructurados existente. El coste es que `dsh-fs` importa la base compartida `HarnessError` de `dsh-llm`; esa dependencia es intencional y se mantiene limitada al vocabulario de errores.

**La rotación de paquetes se paga por adelantado.** La separación en tres paquetes añade código repetitivo antes de que haya más de un backend. Esto es intencional: el acceso al filesystem es un límite probable de sandbox/remoto, y cambiar la API del paquete después de publicar herramientas orientadas al modelo sería más caro.
