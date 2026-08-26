# Agent Note: Seam de capacidad LSP y herramienta de consulta orientada al modelo

Status: implemented

[English](2026-07-15-lsp-capability-seam.md) | Español

## Problema

El harness tiene búsqueda de texto y lectura de archivos, pero ninguna identifica un símbolo de programa. Una coincidencia textual no puede distinguir de forma fiable dos funciones con el mismo nombre, seguir un alias de importación, conectar una interfaz con sus implementaciones ni informar de un tipo inferido. Antes de cambiar código, un agent (agente) carece por tanto de la navegación semántica que un humano obtiene del servidor de lenguaje de un editor.

El soporte de LSP tiene tres propietarios: el modelo necesita un schema de consulta estable, el harness necesita selección de provider y resultados normalizados, y la implementación local necesita comportamiento de proceso, JSON-RPC, espacio de trabajo, sincronización y sistema de archivos. Combinarlos uniría el contrato del modelo a subprocesos locales y obstruiría los providers remotos o nativos del sandbox.

Muchos servidores de lenguaje se comportan mejor cuando el documento consultado se abre con el texto actual. Un cliente de agent compatible debe delimitar ese estado, definir si su lectura de la fuente es una observación del modelo y mantener la instantánea del documento en el mismo espacio de nombres del sistema de archivos que el índice del espacio de trabajo del servidor.

## Decisión

Añadir LSP como seam de capacidad de tres paquetes con una única herramienta de solo lectura orientada al modelo y una implementación genérica de provider local:

1. `@deepseek-ai/dsh-lsp` en `packages/lsp/lsp` es dueño de `ctx.lsp`, del registro y la selección de providers, de las solicitudes y resultados normalizados, del control de ejecución y de los errores LSP estructurados.
2. `@deepseek-ai/dsh-lsp-stdio` en `packages/lsp/lsp-stdio` adapta los servidores de lenguaje stdio configurados al seam. Una instancia de plugin acepta una tabla de servidores con nombre y registra un provider aislado por cada comando y asignación de extensión a id de lenguaje.
3. `@deepseek-ai/dsh-tool-lsp` en `packages/lsp/tool-lsp` es dueño del schema `lsp` orientado al modelo, de la guía del prompt, de la validación de argumentos, de los límites y el formato de los resultados y de la presentación de UI independiente del transporte.

`dsh-lsp-stdio` es un host genérico, no un catálogo de servidores de lenguaje ni un instalador. Los despliegues configuran explícitamente comandos y asignaciones; los preajustes futuros pertenecen a plugins de composición o a overlays de `cordis.yml`.

El modelo y el seam exponen exactamente `goToDefinition`, `findReferences`, `goToImplementation` y `hover`; ningún método JSON-RPC arbitrario se escapa a través de `ctx.lsp`. Estos literales de operación coinciden con los familiares nombres camelCase de Claude Code, mientras que el nombre de la herramienta y el campo `file_path` siguen siendo propiedad del harness.

El prompt posiciona LSP como ayuda de precisión: `Use search/read for ordinary navigation. Use lsp when textual matches are ambiguous or before a change requires precise definitions, implementations, or references.`

## Límites de paquete y titularidad

`dsh-lsp` registra providers por id con marca y asignación de extensión a id de lenguaje. `registerProvider()` reserva atómicamente el id y cada extensión normalizada: una entrada inválida o cualquier conflicto no publica nada, y su disposer libera todas las reservas. Los plugins de provider se registran a través de `ctx.effect()`. La selección es por consulta e independiente del orden; si no hay coincidencia, devuelve un error estructurado de no disponible. La primera versión no tiene selector de glob, id de lenguaje ni ruta explícita, ni capacidades de operación declaradas estáticamente.

El seam expone una única operación `query(request, signal?)` porque ningún campo necesita valores por defecto de implementación: `workspaceRoot` es obligatorio, `languageId` proviene del registro y los consumidores son dueños de los tiempos de espera y los límites de resultado. `query()` selecciona y deriva sin recurrir a sustituciones `??` ocultas, dejando sin resolver ningún spec ejecutable. `dsh-tool-lsp` valida los argumentos del modelo y pasa solo `exec.signal` como un `AbortSignal` puro, igual que en web, y mantiene `dsh-lsp` independiente de `dsh-tools`. La retirada antes de la selección falla como no disponible; la liberación posterior sigue el ciclo de vida de cancelación del provider seleccionado sin redirigir.

La forma del contrato:

```ts
import type { Branded } from '@deepseek-ai/dsh-brand'

type LspOperation = 'goToDefinition' | 'findReferences' | 'goToImplementation' | 'hover'
type LspProviderId = Branded<'LspProviderId'>

interface LspPosition {
  readonly line: number
  readonly character: number
}

interface LspRange {
  readonly start: LspPosition
  readonly end: LspPosition
}

interface LspQueryRequest {
  readonly operation: LspOperation
  readonly filePath: string
  readonly position: LspPosition
  readonly workspaceRoot: string
}

interface LspProviderQuery extends LspQueryRequest {
  readonly languageId: string
}

type LspQueryResult =
  | { readonly kind: 'locations'; readonly locations: readonly { readonly uri: string; readonly range: LspRange }[]; readonly resolvedWorkspaceUri: string }
  | { readonly kind: 'hover'; readonly hover: { readonly contents: string; readonly range?: LspRange } | null }

interface LspProvider {
  readonly id: LspProviderId
  readonly extensionToLanguage: Readonly<Record<string, string>>
  query(request: LspProviderQuery, signal?: AbortSignal): Promise<LspQueryResult>
}

interface LspService {
  registerProvider(provider: LspProvider): () => void
  query(request: LspQueryRequest, signal?: AbortSignal): Promise<LspQueryResult>
}
```

Las claves de las asignaciones se normalizan a minúsculas; las extensiones con punto inicial se toman de la extensión final de `filePath`; los ids de lenguaje solo sincronizan documentos. Las posiciones y rangos del seam son UTF-16 de base cero. `findReferences` siempre incluye declaraciones: los providers lo aplican internamente, la asignación local establece `context.includeDeclaration: true` y los llamadores no reciben ninguna bandera. Las uniones de resultados cerradas normalizan la navegación a ubicaciones y el hover a contenido o `null`; los resultados de navegación llevan la URI canónica del espacio de trabajo del provider para que los consumidores relativicen las URIs de archivo en el espacio de nombres del mundo de ejecución. El seam no expone tipos de protocolo, controles de proceso o documento, ni vía de escape genérica de solicitudes.

`dsh-lsp-stdio` es dueño de la configuración del servidor, de JSON-RPC, del estado de procesos y documentos transitorios y de la traducción de protocolo. Lee a través de `ctx.fs` y lanza a través de `ctx.subprocess`, dependiendo de sus paquetes de Service Definition en lugar de providers concretos; la [decisión de mundo de ejecución portable](2026-07-28-portable-execution-world-consumers.es.md) es dueña de ese emparejamiento. La clave de la tabla de servidores es su id de provider. El plugin resuelve cada ajuste local del servidor antes del registro, revierte los registros anteriores si una asignación posterior es inválida o entra en conflicto, y mantiene un pool de procesos independiente por provider. `dsh-tool-lsp` inyecta en tiempo de ejecución solo `tools`, `lsp` y `systemPrompt`, obtiene el espacio de trabajo de `exec.agent?.session.header.cwd` mediante un helper local al paquete `sessionCwd(exec)` que replica la búsqueda de las herramientas del sistema de archivos, y no importa ningún provider.

## Contrato orientado al modelo

La única herramienta `lsp` acepta:

```ts
interface LspToolInput {
  readonly operation: 'goToDefinition' | 'findReferences' | 'goToImplementation' | 'hover'
  readonly file_path: string
  readonly line: number
  readonly character: number
}
```

`line` y `character` son coordenadas de cursor positivas, de base uno y UTF-16; la herramienta las convierte al `LspPosition` de base cero del seam y convierte de vuelta las ubicaciones renderizadas. `findReferences` incluye declaraciones para que el análisis de impacto no omita el sitio de definición. Provider, id de lenguaje, raíz del espacio de trabajo, límites, tiempo de espera, inicialización y ejecutable quedan fuera de la entrada del modelo.

La herramienta exige `workspaceRoot` del `header.cwd` de la sesión, sin respaldo; su ausencia falla como `LSP_WORKSPACE_REQUIRED` antes de consultar o arrancar. El provider local resuelve las rutas relativas contra esa raíz y acepta rutas absolutas directamente; ambas formas se canonicalizan y se rechazan antes del arranque cuando el destino está fuera del espacio de trabajo canónico.

Las ubicaciones se renderizan como entradas estables `path:line:character` agrupadas por archivo, sin aplicar las reglas de ruta del host del harness. Una URI `file:` válida se convierte en ruta relativa dentro de la URI canónica del espacio de trabajo del provider o en ruta absoluta derivada de la URI cuando queda fuera; las URIs malformadas y las que no son `file:` permanecen verbatim. `maxLocations` tiene como valor por defecto `100` e informa de los elementos omitidos; `maxResultChars` tiene como valor por defecto `16_000` y acota cada resultado renderizado completo, incluidos sus metadatos de truncamiento. Las ubicaciones vacías y el hover `null` son respuestas exitosas sin resultado; las cargas útiles del servidor ausentes o malformadas fallan con errores estructurados `LSP_MALFORMED_RESPONSE`.

El presentador independiente del transporte usa `{ card: 'generic', kind: 'search', title, locations: [{ path: file_path, line }] }` con un `title` de operación/cursor derivado de los argumentos. Como `FileLocation` no tiene character, el seguimiento enfoca la línea de entrada mientras el título conserva el cursor; la presentación permanece pura.

## Titularidad del tiempo de espera

`dsh-tool-lsp` adjunta un presupuesto `timeoutMs` configurable, con valor por defecto `60_000`, a la definición de la herramienta. `dsh-tool-call-timeout-policy` lo aplica y suministra `exec.signal`, que llega a `ctx.lsp.query`; el presupuesto cubre el ciclo de vida completo en cola de abrir/consultar/cerrar y no es configurable por el modelo.

El seam y el provider no añaden plazo de arranque ni de solicitud. Los llamadores que no son herramientas no reciben por tanto ningún tiempo de espera oculto y deben suministrar un `AbortSignal`, usando `deadline()` cuando necesitan un presupuesto.

La liberación del provider ocurre fuera de la ejecución de la herramienta, por lo que `dsh-lsp-stdio` mantiene `shutdownTimeoutMs` (valor por defecto `5_000`) para `shutdown`/`exit` y `killGraceMs` (valor por defecto `2_000`) tanto para la gracia de cancelación de solicitud como para la escalada de SIGTERM a SIGKILL; los mismos límites rigen la limpieza de instancias fallidas. Los valores de temporizador por encima del rango de planificación de `2_147_483_647` ms de Node fallan al cargar. El provider usa `deadline()` y `timeoutOf()`, pero es dueño de la cancelación de solicitudes, de las señales de proceso y de la espera de cierre porque la notificación de tiempo de espera no termina el trabajo.

## Espacio de trabajo, sistema de archivos y sincronización de documentos

`dsh-lsp-stdio` canonicaliza y lee a través de `ctx.fs` en el mundo de ejecución del servidor de lenguaje. Exige que el destino del espacio de trabajo sea un directorio, rechaza las fuentes fuera del espacio de trabajo mediante la contención propiedad del provider, consume `streamText` y aplica `maxDocumentBytes` a medida que llegan los fragmentos; el provider conserva la validación de archivo regular y la decodificación UTF-8 mientras el consumidor de protocolo es dueño de su límite de documento. Fusiona la cancelación del llamador con la liberación del provider en cada operación del sistema de archivos, rastrea las búsquedas del espacio de trabajo antes de que entren en una cola y espera esas búsquedas durante la liberación. No emite `fs/observed`: solo el resultado de LSP es visible para el modelo, por lo que la consulta no satisface la política de leer-antes-de-escribir.

La herramienta `read` no sirve como fuente porque su salida es con ventanas, numerada, visible en el transcript y observada. Leer en `tool-lsp` asignaría además la sincronización específica del provider al consumidor y excluiría a los providers no locales.

El provider local usa una secuencia de apertura transitoria orientada a la compatibilidad en cada consulta. Acepta `textDocumentSync` heredado `Full` o `Incremental`, u opciones con `openClose: true`; las omitidas, `None` o explícitamente incompatibles fallan como no soportadas antes de `didOpen`.

1. Resuelve y contiene la fuente a través de `ctx.fs`, y luego transmite su texto actual por el mismo provider aplicando el límite de bytes del documento.
2. Envía `textDocument/didOpen` con versión `1`, texto completo y el id de lenguaje configurado. Su escritura sigue siendo cancelable; un fallo o cancelación invalida la instancia y espera la terminación acotada del proceso antes de que el pool pueda reutilizarla.
3. Envía la solicitud `textDocument/definition`, `textDocument/references`, `textDocument/implementation` o `textDocument/hover` pedida.
4. Si `didOpen` tuvo éxito, intenta `textDocument/didClose` en `finally` después de que la solicitud se asiente o se aborte. Un fallo de escritura de cierre no reemplaza el resultado o error asentado, pero invalida la instancia y espera la terminación acotada del proceso.

Los documentos se cierran tras cada llamada, por lo que la primera versión no necesita `didChange`, `didSave`, caché de contenido, listener de mutaciones ni LRU de documentos. Una cola cancelable por espacio de trabajo serializa los ciclos de vida de lectura de fuente/apertura/consulta/cierre, de modo que una consulta en espera solo lee los bytes actuales cuando le llega su turno; la instancia también mantiene serializados los ciclos de vida del protocolo. Los espacios de trabajo distintos pueden ejecutarse en paralelo. El índice del espacio de trabajo del servidor sigue siendo responsable de los archivos cerrados alcanzados desde la fuente.

El destino canónico del espacio de trabajo debe ser un directorio. Su clave de destino proporciona la identidad del pool, su ruta de proceso proporciona el cwd y su URI `file:` propiedad del provider proporciona `rootUri` y la única entrada de `workspaceFolders`; los alias comparten una instancia cuando el provider del sistema de archivos los resuelve a una sola clave. Las ubicaciones de resultado pueden ser externas, pero una ruta externa no puede convertirse en fuente de consulta. Un sistema de archivos que no pueda compartir rutas con el provider de subproceso montado es un error de composición, no una razón para otro paquete de LSP.

## Ciclo de vida del servidor local y comportamiento del protocolo

`dsh-lsp-stdio` hace single-flight perezoso de un servidor por (id de provider, destino canónico del espacio de trabajo). Al cargar llama a `ctx.subprocess.resolveExecutable()` con el entorno configurado, fallando antes del registro si no está disponible; la primera consulta lanza a través de tuberías de protocolo en bruto sin shell y con una cola acotada de stderr recopilado. `maxMessageBytes` tiene por defecto `16_000_000`, `maxStderrBytes` `1_000_000` y `maxDocumentBytes` `4_000_000`. Un cuelgue hace fallar la consulta activa sin repetición; una consulta posterior puede reemplazar el proceso. Cada consulta arranca como mucho un proceso, por lo que el MVP no tiene contador de reinicios entre solicitudes.

La inicialización usa `processId: null` porque el cliente y el servidor pueden habitar espacios de nombres de proceso distintos. Anuncia `general.positionEncodings: ['utf-16']`, `workspace: { workspaceFolders: true, configuration: true }`, `textDocument.hover.contentFormat: ['markdown', 'plaintext']` y `linkSupport: true` para definición e implementación, sin registro dinámico. Las capacidades de operación y sincronización devueltas son autoritativas. Un `positionEncoding` del servidor omitido tiene por defecto `utf-16`; cualquier otro valor es un error de protocolo. La configuración puede suministrar opciones de inicialización y respuestas de `workspace/configuration`, pero el cliente rechaza `workspace/applyEdit` y nunca ejecuta comandos ni ediciones.

La navegación asigna `Location` directamente y `LocationLink` desde `targetUri` más `targetSelectionRange`. Las posiciones deben ser enteros no negativos. La normalización del hover acepta solo formas válidas de `MarkupContent` y `MarkedString`, conserva los valores de cadena, renderiza los valores etiquetados con lenguaje como código delimitado y une las matrices con una línea en blanco. La herramienta orientada al modelo aplica `maxResultChars` tras el renderizado.

El aborto llega a cada fase de la consulta y envía `$/cancelRequest` una vez que existe un id. Un servidor que no responde se termina y se espera sin trabajo activo colateral porque la instancia está serializada. La liberación rechaza y cancela el trabajo, intenta un apagado controlado, escala mediante terminación acotada y espera la quiescencia.

## API diferida deliberadamente

Los símbolos se difieren porque necesitan schemas distintos y se solapan con lectura/búsqueda; una futura herramienta de símbolos del espacio de trabajo debe aceptar una consulta de búsqueda. La jerarquía de llamadas se difiere porque el soporte es desigual, y `prepareCallHierarchy` sigue siendo un prerrequisito interno, no una operación de modelo.

Los diagnósticos necesitan reglas separadas de actualidad, acumulación y transcript. Las mutaciones como renombrar, acciones de código y formato requieren herramientas separadas con integración de vista previa, permiso y política de escritura.

El provider confía en su servidor configurado. Su visibilidad del sistema de archivos y su confinamiento de proceso son exactamente los del mundo de ejecución montado; LSP no añade ninguna política de sandbox independiente.

## Alternativas consideradas

**Copiar el schema unificado de Claude Code.** Sus operaciones de cursor validan el caso de uso central, pero los símbolos y la jerarquía de llamadas necesitan argumentos distintos. Copiar las nueve operaciones congelaría superficie especulativa, por lo que el seam se alinea solo con las cuatro consultas semánticas.

**Dejar que los providers registren herramientas.** Los servidores cargados controlarían entonces el schema y los prompts del modelo, impidiendo un contrato estable entre providers locales y remotos.

**Exponer métodos LSP arbitrarios.** Una vía de escape JSON-RPC filtraría cargas útiles de protocolo y admitiría mutaciones o ejecución de comandos sin revisión; la unión de operaciones permanece cerrada.

**Exponer `resolve(request)` / `query(spec)`.** Sin campos con valores por defecto, la resolución solo expondría la selección de provider, y un spec público podría sobrevivir a la liberación o sustitución del provider. Una sola operación mantiene la selección y la invocación atómicas respecto a la vida del registro.

**Envolver la señal en un objeto de contexto de ejecución LSP.** Web pasa un `AbortSignal` puro; envolver este único campo añadiría una asimetría inexplicada. `query()` gana un objeto de contexto solo cuando otro campo lo exija.

**Leer a través de la herramienta `read` orientada al modelo.** Rechazado porque la salida de la herramienta es con ventanas, numerada, visible en el transcript y observada. El provider consume el texto completo transmitido directamente a través del mismo mundo de ejecución `ctx.fs` que usa su subproceso.

**Mantener los documentos abiertos.** Replicar ediciones exige titularidad de versión, `didChange` para todas las rutas, recuperación de HMR, desalojo y reglas de estado obsoleto. Las aperturas transitorias evitan esa máquina de estados del MVP.

**Configurar tiempos de espera por fase.** Los temporizadores anidados crean clasificaciones en competencia y presupuestos nuevos. Un plazo propiedad del llamador cubre el trabajo de consulta; solo el desmontaje fuera de la llamada mantiene límites locales.

**Consultar sin `didOpen`.** Aunque está permitido, el soporte es inconsistente y puede usar estado obsoleto del servidor. La apertura transitoria suministra una instantánea actual explícita.

**Añadir rutas o seleccionar la primera coincidencia.** El orden de registro y el momento del HMR no son semántica de producto, mientras que una tabla de rutas duplicaría la titularidad exclusiva de extensiones. Los solapamientos hacen fallar por tanto el registro.

**Ejecutar consultas concurrentes en una sola instancia.** Si la cancelación falla, terminar el proceso compartido mataría trabajo no relacionado. La serialización por instancia limita ese radio de explosión; las instancias permanecen en paralelo.

**Publicar preajustes o descubrimiento de PATH.** Un catálogo haría que el host genérico fuera dueño de la política de lenguaje, mientras que el descubrimiento no puede inferir argumentos, ids de lenguaje ni inicialización. Los despliegues configuran providers explícitamente; los plugins de composición pueden empaquetar preajustes.

## Pruebas

- Las pruebas de paquete fijan la dirección de dependencia de los tres paquetes, las inyecciones de tiempo de ejecución y la comunicación exclusiva por `ctx.lsp`.
- Las pruebas de herramienta fijan las cuatro operaciones, la validación de coordenadas, los límites y marcadores de omisión configurados, el prompt y la presentación de UI.
- Las pruebas de registro fijan la reserva/liberación atómica, la selección independiente del orden y los errores estructurados de no disponible, liberado, conflicto y operación no soportada.
- Las pruebas de stdio simulado fijan las capacidades exactas de inicialización, las cuatro asignaciones de protocolo, la normalización de `Location`/`LocationLink` y hover, y la asignación de `findReferences` a `references.includeDeclaration`.
- Las pruebas de sincronización fijan la negociación y conversión UTF-16, las formas de `textDocumentSync` soportadas y rechazadas, las escrituras de apertura bloqueadas y fallidas, el apertura/cierre transitorio equilibrado, el fallo de escritura de cierre y el rechazo de respuestas malformadas.
- Las pruebas de tiempo de espera fijan un presupuesto único `TOOL_TIMEOUT`, la cancelación ascendente sin clasificar, la ausencia de plazo oculto de LSP y el desmontaje acotado con espera.
- Las pruebas de ciclo de vida fijan el single-flight de arranque, la serialización del ciclo de vida completo con lecturas de fuente nuevas en cola, el paralelismo entre espacios de trabajo, las colas cancelables, la sustitución tras cuelgue sin repetición, el desmontaje con stdin fallido y la liberación en quiescencia.
- Las pruebas de host del sistema de archivos fijan los requisitos de cwd de sesión, la contención propiedad del provider y el renderizado de URI, las lecturas de documento acotadas, la fuente sin formato y la ausencia del evento `fs/observed`.
- Un e2e de servidor real TypeScript fijado y sin clave ejercita las cuatro operaciones; la configuración ejecutable usa la misma asignación explícita de providers.
- Las instantáneas cubren el schema visible para el modelo, el prompt, los resultados y las omisiones; una prueba de humo del artefacto compilado cubre el encuadre y la limpieza.
- La documentación de paquetes y arquitectura cubre configuración, límites de seguridad y la guía de búsqueda/lectura; el nuevo grupo `packages/lsp/` se añade al bloque de estructura del repositorio de AGENTS.md, a la tabla de grupos de packages/README.md y a architecture.md en el mismo cambio.

## Consecuencias

Los servidores de lenguaje varían en soporte de métodos, interpretación de capacidades y disposición de indexación; LSP no tiene una señal universal de «índice completo». Los servidores sin sincronización de apertura transitoria compatible no se soportan aunque las consultas con documento cerrado funcionen. Los servidores soportados pueden devolver resultados vacíos o parciales, por lo que la herramienta no promete integridad entre servidores. El e2e TypeScript fijado establece un nivel mínimo de compatibilidad, no una afirmación transversal a lenguajes.

Las aperturas transitorias repiten análisis y notificaciones. La serialización por instancia aumenta la latencia con agents en paralelo, y los procesos de espacio de trabajo de larga vida consumen memoria hasta su liberación.

La titularidad de extensiones es exclusiva dentro de un runtime. Dos providers no pueden reclamar ambos `.ts`, ni siquiera con ids de lenguaje distintos; es un límite consciente del MVP. La extensión prevista es un selector configurado en el despliegue por encima de los registros que pueda relajar las reservas exclusivas sin añadir elección de provider a la entrada del modelo ni cambiar `LspProvider.query`.

Las columnas de cursor UTF-16 son exactas para el protocolo pero difíciles de contar para un modelo alrededor de caracteres no BMP. Las posiciones inválidas o fuera de símbolo pueden producir resultados vacíos, por lo que el texto de error y los ejemplos del prompt deben explicar la convención de coordenadas sin fomentar un uso amplio de LSP.

Los providers emparejados de sistema de archivos/subproceso alinean la instantánea de consulta con el índice del servidor, pero no hacen seguro a un servidor de lenguaje de confianza. La contención canónica rechaza las fuentes de consulta fuera del espacio de trabajo en el momento de la resolución, pero la apertura por flujo no añade identidad de identificador estable ante una sustitución concurrente de ruta; el propio servidor recibe la autoridad configurada del mundo de ejecución y puede leer otras rutas o usar cachés.
