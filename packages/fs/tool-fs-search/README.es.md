# @deepseek-ai/dsh-tool-fs-search

[English](README.md) | Español

Las **herramientas de descubrimiento de archivos orientadas al modelo** — `glob`, `grep` — están respaldadas por un binario ripgrep empaquetado, no por métodos del provider `ctx.fs` ni por una instalación `rg` del sistema. Los despliegues Node normales resuelven el binario de plataforma desde `@vscode/ripgrep`; un runtime de un solo archivo pkg resuelve el sidecar `-rg` co-ubicado con el ejecutable y recurre al binario de la dependencia cuando ese sidecar está ausente. El registro es incondicional porque ambos portadores empaquetan ripgrep, de modo que no hay sonda de disponibilidad en tiempo de carga. Cada llamada lanza el binario resuelto a través del seam `ctx.subprocess` con un vector argv fijo (`--no-config` antepuesto para que un `RIPGREP_CONFIG_PATH` host no pueda inyectar un preprocesador `--pre` en el spawn sin confinar; los valores controlados por el modelo son elementos argv simples — no existe capa de shell, por lo que no aplica entrecomillado), analiza la salida `rg` en bruto y devuelve un valor canónico relativo al workdir. El paquete inyecta `tools`, `systemPrompt` y `subprocess` — deliberadamente **no** `fs`; `ctx.spillStore` se lee de forma oportunista con `ctx.get()` porque el spill (volcado) de resultados formateados es opcional.

```ts ignore-check
// A deployment chooses how over-cap glob pages are selected.
await ctx.plugin(LocalSubprocessRuntime)                     // @deepseek-ai/dsh-subprocess-local
await ctx.plugin(ToolFsSearch, { sampleOverCapGlobResults: false })
// Optional: a spill backend makes capped results fully recoverable.
await ctx.plugin(LocalSpillStore)                           // @deepseek-ai/dsh-spill-local
```

Por qué con spawn: el descubrimiento local del workspace es naturalmente un flujo de trabajo `rg` respaldado por procesos, y poner la búsqueda en `ctx.fs` obligaría a todo backend de sistema de archivos a incorporar una API de búsqueda. El seam de subprocesos es propietario de la ejecución del spawn, la terminación del árbol de procesos, el saneamiento del entorno y la captura de salida acotada; este paquete es propietario de los schemas, la validación de argumentos, la construcción de argv, el análisis, la retención, el spill de resultados formateados y la declaración de tiempo de espera. Las herramientas nunca exponen un trabajo en segundo plano — la llamada solo devuelve después de que `rg` salga, sea terminado por el tiempo de espera cooperativo, sea abortado o falle.

## Requisito de despliegue: sin rg host, workdir/sistema de archivos co-ubicados

Los despliegues Node reciben el paquete de plataforma `@vscode/ripgrep` en los objetivos macOS, Linux y Windows x64/arm64 compatibles. Las wheels de Linux y macOS del SDK Python copian el binario nativo del objetivo junto al runtime de un solo archivo como `<runtime>-rg`; `deepseek_harness_runtime.bundled_runtime_path()` rechaza una wheel incompleta antes del lanzamiento. Ningún portador requiere una instalación `rg` host. Las rutas devueltas se muestran relativas al workdir resuelto (el cwd de sesión del agent (agente) llamante si existe; si no, `process.cwd()`) y solo se pueden volver a leer con `read` cuando ese workdir y la raíz del sistema de archivos son el mismo workspace. Ese requisito de co-ubicación no conlleva validación entre servicios en tiempo de ejecución; la búsqueda en sistema de archivos remoto o virtual espera un contrato de workspace compartido o un backend de búsqueda específico del provider.

## Configuración

`sampleOverCapGlobResults` es obligatorio y no tiene valor de respaldo; los despliegues eligen explícitamente el contrato de ordenamiento por encima del límite. Las claves restantes son límites de búsqueda opcionales con los valores por defecto siguientes.

| Clave | Valor por defecto | Significado |
|---|---|---|
| `sampleOverCapGlobResults` | ninguno (obligatorio) | `true` muestrea una página `glob` por encima del límite entre las entradas de nivel superior; `false` conserva el encabezado ordenado por hora de modificación. Cuando el spill formateado tiene éxito, ambos modos preservan la lista ordenada completa en ese artefacto. |
| `globMaxResults` | `100` | Máx. de rutas que una llamada `glob` muestra en línea (coincide con el límite `GlobTool` de Claude Code). Un resultado dentro del límite permanece completo y ordenado por hora de modificación. |
| `grepMaxMatches` | `250` | Máx. de coincidencias planas que una llamada `grep` retiene en línea (coincide con `head_limit` de `GrepTool` de Claude Code); las coincidencias posteriores van al artefacto de spill formateado. |
| `grepMaxLineBytes` | `2000` | Límite de bytes por vista previa de línea coincidente; el corte preserva los límites UTF-8 y se marca `(line truncated)`. |
| `rawOutputMaxBytes` | `20000000` | Máx. de salida `rg` stdout en bruto completa que una búsqueda analizará (coincide con el búfer ripgrep en bruto de Claude Code); una salida en bruto mayor falla con `SEARCH_RAW_OUTPUT_OVERFLOW`. |
| `timeoutMs` | `30000` | Presupuesto cooperativo de llamada de herramienta adjunto a ambas definiciones de herramienta, aplicado por `@deepseek-ai/dsh-tool-call-timeout-policy` a través de `exec.signal`; la escalada de terminación del seam de subprocesos es el kill forzado. |
| `graceMs` | `3000` | Margen de gracia positivo de escalada de terminación que el seam de subprocesos concede más allá de `timeoutMs` antes de que la búsqueda falle como `SEARCH_ABORTED`; no puede superar [`MAX_TIMER_DELAY_MS`](../../util/timeout/README.es.md). |
| `stderrMaxBytes` | `65536` | Presupuesto de cola de diagnóstico para el stderr de `rg`, capturado mediante la disposición de recopilación del seam de subprocesos; una lectura con pérdida conserva solo la cola (marcada `[stderr truncated]`). |

## Herramientas

| Herramienta | Argumentos | Comportamiento |
|---|---|---|
| `glob` | `pattern`, `path?` | `rg --files --glob <pattern> --sort=modified --no-ignore --hidden` más exclusiones de metadatos de VCS (`.git`, `.svn`, `.hg`, `.bzr`, `.jj`, `.sl`). `path` es una raíz de búsqueda de directorio opcional; si se omite, significa el workdir resuelto. Devuelve una ruta de ARCHIVO por línea; `rg --files` nunca emite entradas de directorio. El patrón conserva la semántica de ripgrep: sin `/` coincide con el basename a cualquier profundidad, de modo que `*` coincide con todo el árbol. Los resultados completos permanecen ordenados por hora de modificación; la presentación por encima del límite sigue `sampleOverCapGlobResults`. |
| `grep` | `pattern`, `path?`, `include?` | Análisis `rg --json` orientado a líneas (sin ambigüedad de división por dos puntos). `pattern` es una regex de ripgrep; `path` es un objetivo de archivo o directorio opcional; `include` es UN filtro glob positivo — un valor de lista separada por comas o negado (`!…`) se rechaza de antemano (la alternancia con llaves como `*.{ts,tsx}` está bien). Devuelve las coincidencias agrupadas por archivo como `Line N: <preview>`. |

Los presupuestos rutinarios quedan fuera del schema orientado al modelo (sin `head_limit`/`offset`/`case_insensitive`/modos de salida): un modelo que necesita contexto circundante lee el archivo coincidente con `read`; uno que necesita resultados posteriores sigue la pista de recuperación del localizador de spill devuelto.

## Dos presupuestos, dos artefactos

El stdout y el stderr de `rg` en bruto son detalles de transporte internos. Cada búsqueda solicita presupuestos de modo de recopilación al seam de subprocesos — stdout completa dentro de `rawOutputMaxBytes` y una cola de diagnóstico de `stderrMaxBytes` — sin archivos de spill en ninguna de las dos transmisiones (la herramienta nunca lee una ruta de spill en bruto). Si el seam aún informa una lectura de stdout con pérdida, la búsqueda falla con `SEARCH_RAW_OUTPUT_OVERFLOW` y le dice al modelo que reduzca la consulta; una lectura de stderr con pérdida solo marca el extracto de diagnóstico `[stderr truncated]`. Un `glob` exitoso conserva la raíz de búsqueda mostrada y cada ruta adquirida en `{ root, paths }`; cuando el muestreo está habilitado, `root` permite que el renderizador Native agrupe una ruta de búsqueda explícita relativa o absoluta por entradas bajo esa raíz, en lugar de por su prefijo de workdir. `grep` conserva cada `{ path, lineNumber, line }` adquirido en `{ matches }`. Los límites de elementos en línea y de vista previa por línea se aplican solo en el renderizador Native. Para una llamada directa de superficie con más resultados lógicos que el límite en línea, un intento de mejor esfuerzo posterior a la política guarda la vista previa formateada completa mediante `ctx.spillStore.saveText()` y reemplaza solo la presentación con la página configurada más el localizador. Los dispatch de Nested Code omiten ese spill porque su valor canónico completo no entra en el contexto del modelo. Un spill ausente/fallido conserva la página en línea e informa de que el resultado completo no se pudo guardar — nunca un `isError`.

## Errores

Los fallos de búsqueda transportan el `SearchError` propiedad del paquete (una subclase de `HarnessError`), expuesto como `{ name, code }` en los resultados `isError`: `SEARCH_INVALID_PATTERN` (ripgrep rechazó la regex/el glob), `SEARCH_FAILED` (un lanzamiento `rg` fallido, un objetivo inaccesible, un kill por señal, una salida `--json` malformada), `SEARCH_RAW_OUTPUT_OVERFLOW` (salida en bruto por encima de `rawOutputMaxBytes`, o aún con pérdida tras el presupuesto de captura de stdout solicitado) y `SEARCH_ABORTED` (tiempo de espera cooperativo de herramienta o cancelación del llamador). La semántica de salida de ripgrep es propiedad de la herramienta: la salida 0 es éxito con resultados, la salida 1 es una búsqueda vacía exitosa (`No files found` / `No matches found`), y solo las demás salidas son fallos. Los errores de argumentos del modelo (patrón en blanco, un `include` con valor de lista) siguen siendo errores de argumentos de herramienta ordinarios.

## Experiencia del modelo

### Prompt de sistema

#### Lo que ve el modelo

Cada solicitud en el ámbito de registro de este plugin contiene la guía de glob y grep registrada de forma independiente que se muestra a continuación. Las restricciones de herramientas con ámbito de agent pueden ocultar cualquiera de los dos schemas sin eliminar su sección de prompt.

##### Guía de glob con `sampleOverCapGlobResults: true`

```markdown
Use the glob tool — not shell find — to discover files by path pattern. A pattern with no "/" matches basenames at any depth, so "*" matches every file in the tree rather than its top level. Results are files only, never directories, and include hidden and ignored files: a result that fits comes back in modification-time order, while a larger one is sampled across top-level entries, so it spans the tree instead of one subtree.
```

##### Guía de glob con `sampleOverCapGlobResults: false`

```markdown
Use the glob tool — not shell find — to discover files by path pattern. A pattern with no "/" matches basenames at any depth, so "*" matches every file in the tree rather than its top level. Results are files only, never directories, and include hidden and ignored files: a result that fits comes back in modification-time order, while a larger one keeps the modification-time-ordered head.
```

##### Guía de grep

```markdown
Use the grep tool — not shell grep or rg — to search file contents. Use read on a matched file when you need surrounding context.
```

#### Efecto en tokens

Coste de guía fijo por solicitud mientras las herramientas están registradas; la elección de muestreo obligatoria selecciona una variante de glob.

#### Efecto en la caché KV

Estable respecto al prefijo mientras el ámbito del plugin, la elección de muestreo y el texto de guía no cambien. La activación, la eliminación o el cambio de la elección pueden invalidar la reutilización de esta sección de prompt.

### Schemas de herramientas

#### Lo que ve el modelo

La descripción de glob declara el ordenamiento configurado por encima del límite. Los [schemas `glob` y `grep` generados](../../../docs/tool-catalog.es.md#deepseek-aidsh-tool-fs-search) usan `sampleOverCapGlobResults: true`; las herramientas se registran incondicionalmente.

#### Efecto en tokens

Coste de schema fijo en cada solicitud donde las herramientas son visibles.

#### Efecto en la caché KV

Estable respecto al prefijo mientras la visibilidad y las definiciones de las herramientas no cambien. El ciclo de vida del registro o las restricciones de ámbito pueden invalidar la reutilización desde el primer token de schema modificado.

### Resultados y avisos de spill

#### Lo que ve el modelo

`glob` devuelve una ruta por línea; `grep` agrupa las coincidencias `Line <line>: <preview>` bajo cada ruta. Las búsquedas vacías devuelven `No files found` o `No matches found`. Un resultado con límite termina con su recuento de omisiones más el localizador de spill y la pista de recuperación del backend, o dice que el resultado completo no se pudo guardar. Con `sampleOverCapGlobResults: true`, una página `glob` por encima del límite toma rutas en round-robin entre las entradas inmediatamente bajo la raíz de búsqueda real, y el pie de página declara la base muestreada y cuántas entradas de nivel superior alcanzó; cuando no puede alcanzarlas todas, el pie de página le dice al modelo que reduzca `path`. Con `false`, la página es el encabezado ordenado por hora de modificación y conserva el pie de página normal de resultado con límite. Un resultado que cabe en línea no se toca, y un resultado muestreado plano también conserva el pie de página normal porque su muestra equivale al encabezado por hora de modificación. El artefacto de spill siempre contiene la lista completa en orden por hora de modificación.

#### Efecto en tokens

Las rutas y coincidencias en línea están acotadas por `globMaxResults`, `grepMaxMatches` y `grepMaxLineBytes`; la llamada y el resultado retenido permanecen en el historial hasta la compactación.

#### Efecto en la caché KV

Solo anexión; el contenido recién visible sigue el prefijo de solicitud reutilizable y no invalida las entradas existentes de la caché KV.

### Errores de herramientas

#### Lo que ve el modelo

Los fallos se normalizan como `Error: <message>` con metadatos estructurados `SEARCH_INVALID_PATTERN`, `SEARCH_FAILED`, `SEARCH_RAW_OUTPUT_OVERFLOW` o `SEARCH_ABORTED` para los llamadores.

#### Efecto en tokens

Solo una llamada que falla añade estos tokens retenidos.

#### Efecto en la caché KV

Solo anexión; el contenido recién visible sigue el prefijo de solicitud reutilizable y no invalida las entradas existentes de la caché KV.

## Limitaciones conocidas y trabajo diferido

- **La búsqueda y el acceso a archivos no tienen prueba de workspace compartido** — las rutas devueltas solo se pueden volver a leer cuando el workdir y la raíz del sistema de archivos denotan el mismo workspace; el paquete no realiza validación entre servicios en tiempo de ejecución.
- **El binario empaquetado se fija a la versión de la dependencia** — los despliegues Node usan la versión seleccionada por `@vscode/ripgrep`; los runtimes Python de un solo archivo copian esa versión nativa del objetivo en el sidecar `-rg` obligatorio. Una plataforma no compatible o una instalación corrupta falla con `SEARCH_FAILED`, mientras que el paquete de runtime Python rechaza un sidecar ausente antes del lanzamiento. Los sistemas de archivos remotos o virtuales necesitan un workspace co-ubicado u otro Consumer de búsqueda.
- **Los schemas exponen una única página acotada** — la paginación por offset, los conmutadores de modo de mayúsculas, los modos de salida alternativos y el descubrimiento respaldado por provider permanecen fuera de este paquete; la salida completa con límite requiere un backend de spill.
- **El muestreo, cuando está habilitado, agrupa solo por el primer segmento de ruta bajo la raíz de búsqueda** — una página `glob` por encima del límite equilibra entre esas entradas de nivel superior, de modo que un resultado concentrado más abajo (un directorio ocupado dentro de un árbol por lo demás uniforme) aún se muestra de forma desigual por debajo de ese nivel; el equilibrio recursivo está diferido.
