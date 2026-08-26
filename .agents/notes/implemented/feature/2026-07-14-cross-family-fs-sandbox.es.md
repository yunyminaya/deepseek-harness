# Agent Note: Sandbox de archivos entre familias — un único hogar de política, un provider de fs con sandbox y paridad de escalada de fs

Status: implemented

[English](2026-07-14-cross-family-fs-sandbox.md) | Español

## Problema

`SandboxMode` reclama los efectos sobre archivos, pero originalmente solo `ctx.shell` lo aplicaba. Las herramientas de fs (`write`/`edit`) mutan el sistema de archivos del host en proceso a través de `ctx.fs`, donde un envoltorio de argv del SO es mecánicamente irrelevante — [la Agent Note del sandbox](2026-07-06-sandbox.es.md) § Herramientas en proceso registra este hecho y dejó la aplicación entre familias como una fase diferida con una pregunta abierta: si la aplicación en proceso se mantiene por seam o se convierte en una capacidad uniforme del agent harness (marco de trabajo para agentes). Esta Agent Note es esa fase, y la responde: un único hogar de política compartido, con aplicación por seam en la altitud correcta de cada familia.

El hueco no tenía forma de solo lectura. El modo de producto de un coding agent confinado es `workspace-write`: bash ya puede escribir bajo la raíz del workspace mientras todo lo demás queda denegado, así que una aplicación de fs que solo pudiera denegarlo todo sería estrictamente peor que desactivar las herramientas de fs — el modelo intentaría un `write` dentro del workspace, sería denegado y aprendería a desviarse por heredocs de `bash`. La aplicación entre familias habla, por tanto, la escalera completa de modos, incluido el juicio de contención de rutas que `workspace-write` exige (destinos canónicos; escapes por `..`/symlink/ruta absoluta) y la misma palanca de escalada que lleva bash.

Una segunda familia aplicadora también expuso un problema de propiedad en el diseño original. El valor por defecto del despliegue (`mode` + `workspaceRoot`) se configuraba en `dsh-bash-sandbox`, y el evento de anulación por sesión era `shell/sandbox-mode`, plegado y escrito por el kit de modo de sesión de `dsh-shell`. Con fs aplicando la misma política, o bien fs lee la configuración y los eventos de bash (una familia de capacidades dependiendo de la configuración de plugin de un hermano), o bien cada familia lleva su propia copia — y dos copias de `workspaceRoot` derivan exactamente hacia el mundo dividido contra el que advierte el RFC del sandbox: bash confinado a una raíz mientras fs cerca otra.

## Decisión

Tres piezas coordinadas, todas compuestas desde el `cordis.yml` hoja, ninguna toca `agent-loop`.

### `ctx.sandboxPolicy`: un único hogar para el modo y la raíz del workspace

`packages/sandbox/sandbox-policy/` (`@deepseek-ai/dsh-sandbox-policy`) registra `ctx.sandboxPolicy`, el único propietario de la política de sandbox del despliegue:

- `Config`: `mode` (la unión cerrada `SandboxMode`, por defecto `read-only`) y `workspaceRoot` (por defecto el cwd del proceso, resuelto absoluto). Una mala configuración falla con ruido en la carga.
- El evento de anulación por sesión `sandbox/mode`, con su pliegue puro (`effectiveSandboxMode(events)`), su ruta de escritura (`setSandboxMode(session, mode)`) y `SANDBOX_MODES`. El evento es estado de política — lo consumen dos familias — así que vive aquí, no en el seam de ninguna de las dos capacidades. Su forma y su semántica de solo registro siguen el precedente de `approval/*`.
- `resolve({ session?, mode? })`, que devuelve una `SandboxExecutionPolicy` completa por llamada: modo aprobado explícito > pliegue de la sesión > `defaultMode`, y el cwd inmutable de la sesión > el `workspaceRoot` configurado como respaldo.
- Los accesores `defaultMode` / `workspaceRoot` se conservan como respaldos del despliegue y como hecho de anuncio de capacidad.

`dsh-bash-sandbox` no lleva configuración de sandbox propia — inyecta `sandboxPolicy` y usa su respaldo de despliegue solo para llamadas directas. `dsh-tool-bash` y `dsh-tool-fs` pasan la sesión activa a `ctx.sandboxPolicy.resolve()`, de modo que ambos reciben el mismo modo efectivo y la misma raíz de cwd en cada llamada; los presets de `dsh-permission-presets` y el puente ACP escriben a través del setter reubicado. Los seams que poseen la ejecución de bash y fs siguen sin sesión — la dependencia de sesión vive en el paquete de política y en los consumidores de herramientas.

### `dsh-fs-sandbox`: aplicación dentro del provider

`packages/fs/fs-sandbox/` (`@deepseek-ai/dsh-fs-sandbox`) refleja la división `bash-local`/`bash-sandbox`: `SandboxedFileSystem extends LocalFileSystem`, registrado como `ctx.fs`, inyectando `sandboxPolicy`. Las lecturas (`resolve`/`stat`/`readText`/`streamText`/`listDir`) pasan intactas — todos los modos permiten leer. Las dos mutaciones aplican por modo antes de delegar en la escritura atómica heredada:

- `read-only` deniega `writeText`/`editText` directamente.
- `workspace-write` cerca el destino canonizado contra el conjunto de raíces escribibles — `writableRoots(policy)` en `dsh-sandbox`: la raíz del workspace más las áreas temporales de la plataforma (`/tmp`, `os.tmpdir()`), cada una con realpath — el MISMO conjunto que concede el perfil Seatbelt, de modo que la cerca de fs es el cuarto dialecto de un único significado de modo junto a los perfiles bwrap/Landlock/Seatbelt, y no pueden surgir asimetrías como «la herramienta de escritura no puede escribir en `/tmp` pero bash sí». Las grafías canónicas toman una vía rápida de contención léxica; cuando Windows expone un directorio con distinto uso de mayúsculas o grafías de nombre largo/8.3, un recorrido de ancestros compara la identidad del sistema de archivos en lugar de debilitar la frontera a suposiciones textuales de prefijo. El destino se re-canoniza (`resolve` aplica realpath al ancestro existente más profundo) inmediatamente antes de delegar, de modo que un symlink de ancestro intercambiado desde que la herramienta lo resolvió queda detectado.
- `danger-full-access` delega sin cerca.

Una denegación es el `FS_SANDBOX_DENIED` estructurado que lleva el modo efectivo — distinto de `FS_PERMISSION_DENIED` (un EACCES del host es el mundo negándose; esto es la política negándose). Sin inferencia de texto: una cerca en proceso sabe exactamente qué denegó. El portador por llamada es una `SandboxExecutionPolicy` final opcional en `writeText`/`editText` (el gemelo de `ShellExecRequest.sandboxPolicy` en el sistema de archivos); el seam sigue sin sesión, y el backend local desnudo la ignora. `FileSystem.sandboxMode` es el hecho de capacidad (`undefined` en la base y en `fs-local`, el valor por defecto en `SandboxedFileSystem`), de modo que la capa de herramientas anuncia la escalada a partir de la verdad de composición.

El modelo de amenaza se declara en el README del paquete: una cerca de política en código de confianza sobre rutas controladas por el modelo, no una frontera de kernel — las operaciones son las propias del seam, solo la ruta destino es no confiable, así que canonizar-y-luego-contener es la respuesta completa a esta superficie (el precedente de «contención, no frontera de seguridad» de `code-runtime`). El aislamiento de CÓDIGO no confiable a nivel de kernel sigue siendo trabajo de `ctx.shell`. La carrera residual resolver-a-syscall se estrecha con la re-canonización en el sitio y solo se elimina con primitivas de plataforma (`openat2` `RESOLVE_BENEATH`) que no merecen aquí su coste de portabilidad.

### Paridad de herramientas: un marcador de denegación, un flujo de escalada

`dsh-tool-fs` resuelve la política completa de la sesión activa sobre cada mutación y mapea `FS_SANDBOX_DENIED` al marcador que el modelo ya conoce de bash: `[sandbox: file access denied under <mode> mode]`. Cuando `ctx.fs.sandboxMode` informa un modo confinante en el registro, `write` y `edit` anuncian los mismos campos `sandbox_permissions` + `justification`, enseñan el mismo reintento en el mismo turno y resuelven la misma solicitud de `ctx.approval` antes de ejecutar — los cuatro resultados y sus textos verbatim de cierre en fallo tomados de [la Agent Note del sandbox](2026-07-06-sandbox.es.md) § Escalada (ensanchamiento estricto comprobado en la ejecución contra el modo efectivo de la llamada; una concesión cambia solo el modo de esa llamada y conserva la raíz de su sesión; sin nuevos eventos de sesión).

Las piezas compartidas viven en `dsh-sandbox`, que posee los tipos de modo: `WIDER_MODES`, la enumeración de destinos de escalada, la validación de emparejamiento de argumentos, los constructores de marcadores de denegación/pista y `approveEscalation` — la coreografía ordenada de cierre en fallo. `approveEscalation` toma un aprobador ESTRUCTURAL mínimo (`EscalationApprover`, genérico sobre los tipos de agent y call-id), no el tipo de servicio de aprobación, de modo que `dsh-sandbox` no gana dependencia de los paquetes de aprobación ni de agent: cada herramienta pasa su propio `ctx.approval`, agent, call id y nombre de herramienta como ingredientes. `dsh-tool-bash` y `dsh-tool-fs` usan ambos estas piezas; el gate de duplicación entre archivos mantiene honesta la fuente única.

La composición [`examples/acp-agent`](../../../../examples/acp-agent/cordis.yml) carga `dsh-sandbox-policy` y `dsh-fs-sandbox`, mueve la configuración de `mode`/`workspaceRoot` a la entrada de política y elimina el antiguo gating que desactivaba la pila de fs bajo modos confinados; `fs-observation-policy` (leer-antes-de-editar) se compone ortogonalmente encima. El system prompt sigue sin declarar modo de sandbox — el marcador enseña la frontera en el momento en que importa, según la evidencia en vivo de la Agent Note del sandbox.

### El punto de aplicación: el provider, no una puerta de intención

El boceto original entre familias de la Agent Note del sandbox ponía la aplicación de fs en los eventos `fs/write-intent`/`fs/edit-intent`. Esta Agent Note aplica en el provider en su lugar, por dos hechos mecánicos: los slots de intención son de decisión única con primero-que-gana (ocupados por `dsh-fs-observation-policy`, cuyo contrato llama mala configuración a un segundo decididor), y los eventos de intención los despacha solo `dsh-tool-fs` — un llamador directo de `ctx.fs` (un plugin montado en cordis, una herramienta personalizada) los sortea, mientras que la aplicación a nivel de provider cubre a todo llamador por construcción.

### Fuera de alcance

- **Política de red para `ctx.web`** — `SandboxMode` reclama solo efectos sobre archivos; un mando de red solo-web mientras `curl` de bash corre libre sería una frontera falsa. Retomar cuando un backend de bash aplique red (bwrap `--unshare-net`, Landlock ABI v4+).
- **El consumidor `subagent-acp`** — fase diferida sin cambios del RFC del sandbox.
- **Raíces escribibles adicionales dentro de una sesión** — la política resuelta lleva un `SessionHeader.cwd` primario; `additionalDirectories` de ACP sigue siendo un puente y un diseño de política separados.
- **Un runtime de sandbox uniforme por herramienta** — sigue rechazado por las razones del RFC del sandbox.

## Alternativas consideradas

- **Aplicar en los eventos de intención `fs/*` (el boceto original de la Agent Note del sandbox)** — rechazado por los dos hechos mecánicos de § El punto de aplicación: primer-que-gana de slot único ya ocupado, y un bypass para llamadores directos de `ctx.fs`. La aplicación a nivel de provider cubre a todo llamador y refleja la forma de intercambio-de-implementación de bash.
- **Aplicar en `tools/pre-execute`** — rechazado: el listener ve la cadena de ruta cruda del modelo antes de `resolve()`, así que reimplementaría el valor por defecto del cwd y la canonización de symlinks y aun así correría contra la resolución real. Descalificante para `workspace-write`, un juicio sobre rutas canónicas.
- **Comprobaciones en línea en `dsh-tool-fs`** — rechazado: cubre solo la ruta de la herramienta (mismo bypass que los eventos de intención) y duplica el conocimiento de resolución una capa por encima de donde el destino canónico ya existe.
- **Una bandera `mode` en `dsh-fs-local` en lugar de un backend hermano** — rechazado: el hecho de capacidad debe ser verdad de composición como lo es `dsh-bash-local` frente a `dsh-bash-sandbox`; una bandera de configuración hace condicional el anuncio de la herramienta, y la familia bash ya establece la forma de paquete hermano.
- **Mutaciones de fs aplicadas por kernel mediante un subproceso helper confinado** — rechazado: un proceso por escritura; la sección crítica leer-concordar-escribir de `editText` tendría que moverse entera al hijo para seguir siendo atómica; y la superficie de amenaza (operaciones de confianza, argumento de ruta no confiable) no necesita un kernel — la cerca en código de confianza es la respuesta completa, mientras que el aislamiento de código no confiable se queda en `ctx.shell`.
- **Configuración de política por familia con comprobación de consistencia en carga** — rechazado: dos hogares para un hecho, remendados por una comprobación que debe enumerar a toda familia aplicadora futura; el servicio de política hace la deriva inexpresable en lugar de detectada.
- **Mantener el evento de anulación en `dsh-shell` como `shell/sandbox-mode`** — rechazado: el evento es estado de política consumido por dos familias; dejarlo con nombre de bash obliga a `dsh-fs-sandbox` a depender del vocabulario de bash. Pre-lanzamiento, el renombrado es un movimiento en el mismo cambio con re-registros de instantáneas, sin shims.
- **Coreografía de escalada importada de los paquetes de aprobación/agent a `dsh-sandbox`** — rechazado: invertiría el apilamiento (un paquete de vocabulario base dependiendo de paquetes de UI/agent). El aprobador estructural mantiene la lógica con fuente única en `dsh-sandbox` mientras las dependencias se quedan en la capa de herramientas que ya las tiene.
- **Un objeto consolidado de opciones de mutación en el seam de fs** (la forma esbozada primero para el portador por llamada) — rechazado por fricción: divide `signal` entre una bolsa de opciones para mutaciones mientras las lecturas lo mantienen posicional. Una `SandboxExecutionPolicy` final opcional coincide con el patrón de llevar-e-ignorar de bash y mantiene `signal` simétrico en todo el seam.
- **Concesiones adicionales de raíces escribibles en `SandboxPolicy` ahora** — diferido sin cambios: `writableRoots()` se deriva hoy del significado del modo; las concesiones ad hoc son una cuestión de alcance de escalada que el RFC del sandbox dejó abierta.

## Consecuencias

Lo que se envió — los niveles de § Pruebas mantienen cada uno:

- Bajo `read-only`, `write`/`edit` devuelven el marcador `[sandbox: file access denied under read-only mode]` y el disco queda intacto; `read`/`listDir` se comportan idénticamente a `dsh-fs-local`.
- Bajo `workspace-write`, las mutaciones aterrizan bajo la raíz del workspace y las áreas temporales y quedan denegadas fuera; la matriz de contención — travesía de `..`, rutas absolutas fuera, un directorio con symlink preexistente dentro que apunta fuera, un archivo nuevo creado bajo tal symlink y grafías de raíz equivalentes por alias — deniega cada escape admitiendo la misma identidad de directorio en discos reales.
- Una mutación de fs denegada reintentada una vez con `sandbox_permissions` + `justification` pasa por la cadena de aprobación compuesta; una concesión ejecuta exactamente esa llamada bajo el modo más amplio y la escritura aterriza; rechazada/cancelada/no disponible producen cada una su texto verbatim de cierre en fallo y no mutan nada.
- Un único interruptor de preset `permission` gobierna ambas familias: después de que una sesión cambie de modo, la siguiente llamada de bash y la siguiente mutación de fs honran ambas el nuevo modo desde el mismo pliegue `sandbox/mode`.
- Sesiones concurrentes con distintas raíces de cwd llevan políticas distintas a través de las mismas instancias de servicio; ninguna familia cachea la raíz de una sesión para la siguiente llamada.
- Un `ctx.fs.writeText` directo sin sello por llamada queda confinado al valor por defecto del despliegue.
- Los campos de escalada en `write`/`edit` existen exactamente cuando el `ctx.fs` montado confina; ausentes bajo `dsh-fs-local`.
- `agent-loop` queda intacto — todo viaja en `ctx.sandboxPolicy`, el seam `ctx.fs`, la fusión de `SessionEventMap` y el pipeline de ejecución de herramientas.

Costes y límites aceptados:

- **La cerca de fs es una frontera de política, no de kernel.** Su superficie de amenaza son rutas elegidas por el modelo, no procesos host adversarios; el TOCTOU residual resolver-a-syscall queda estrechado, no eliminado, y el README lo dice. Las fronteras de kernel siguen siendo de bash.
- **`dsh-bash-sandbox` gana una dependencia dura de `ctx.sandboxPolicy`.** Cada composición con sandbox añade una entrada `cordis.yml` o falla con ruido en la carga — el movimiento de cimientos pre-lanzamiento previsto; los ejemplos se actualizan en el mismo cambio.
- **La paridad cerca-vs-runner se deriva, no se afirma.** La cerca de fs y el perfil Seatbelt toman ambas su conjunto escribible de `writableRoots`, y una prueba de paridad fija los conjuntos; un perfil de runner que cambiara su conjunto escribible sin esa función derivaría.
- **El marcador y la enseñanza de escalada sirven ahora a dos familias.** Un cambio de redacción es una edición coordinada detrás de un único constructor en `dsh-sandbox`; el gate de duplicación y las instantáneas fijadas lo mantienen con fuente única, al coste de que fs y bash no pueden divergir deliberadamente en el fraseo sin partir el constructor.

## Pruebas

- Unit: `dsh-sandbox` fija la escalera de escalada, los constructores de marcadores, la validación de emparejamiento de argumentos y la secuencia ordenada de cierre en fallo de `approveEscalation` (sin ensanchamiento, sin aprobación, sin agent, cada resultado), además de `writableRoots`/`canonicalPath`. `dsh-sandbox-policy` fija el respaldo del despliegue, la resolución de modo/raíz de sesión, la precedencia del modo explícito, el pliegue/setter, el rechazo de modo en carga y la seguridad HMR (hot module replacement). `dsh-fs-sandbox` fija la cerca por política y la matriz de contención (dentro, área temporal, absoluto-fuera, `..`, directorio con symlink hacia fuera, archivo nuevo bajo uno, ruta-igual-a-raíz, raíz-del-sistema-de-archivos, raíz-terminal-en-separador y grafía equivalente por alias) en un sistema de archivos real, además de la anulación por llamada y la seguridad HMR. `dsh-tool-fs` fija el gating de anuncio, la resolución completa de política, el mapeo del marcador de denegación y la matriz completa de escalada (concesión, rechazo, sin-servicio, sin-agent, emparejamiento, guardia de no-confín). `dsh-tool-bash`, `dsh-bash-sandbox` y `dsh-permission-presets` consumen el mismo kit de política.
- e2e sin clave: un contexto Cordis real crea dos agents con distintas raíces de cwd de sesión, ejecuta de forma concurrente las herramientas de bash y fs enviadas, y verifica en el mundo que las escrituras de proyecto propio aterrizan mientras ambas escrituras entre proyectos quedan denegadas.
- Instantánea: el ejemplo acp-agent compone `dsh-sandbox-policy` + `dsh-fs-sandbox`; el encabezado fijado lleva los campos de escalada de fs y el nombre de evento `sandbox/mode`, re-registrado una vez.
