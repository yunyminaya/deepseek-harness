# Agent Note: Política de spill de salida de herramientas

Status: implemented

[English](2026-07-08-tool-output-spill-files.md) | Español

## Problema

Las salidas de las herramientas necesitan vistas previas acotadas orientadas al modelo, pero algunos resultados sobredimensionados siguen siendo útiles más adelante. El cuerpo de una página descargada o una respuesta de herramienta verbosa no debería consumir por completo la siguiente solicitud del modelo, pero el modelo debería poder inspeccionar después el resultado completo formateado con las herramientas de lectura de archivos existentes.

Antes de este cambio el comportamiento era desigual. `dsh-bash-local` ya escribía los flujos completos de stdout/stderr en archivos de spill temporales privados cuando su cola en memoria se desbordaba, pero los resultados de texto ordinarios de las herramientas se devolvían en línea salvo que la herramienta implementara su propio límite a mano. La [biblioteca de retención de resultados de herramientas](2026-07-06-tool-result-retention-library.es.md) es la dueña de la mecánica de vista previa, pero no es dueña del almacenamiento ni de una política del pipeline de ejecución que aplique esa mecánica a los resultados finales de las herramientas.

La forma coincide con el diseño de la política de timeout: el autor de una herramienta declara un valor canónico más un renderer Native, y un plugin de política hace cumplir el presupuesto de contexto por defecto del despliegue sobre el contenido renderizado. El spill temprano específico de la herramienta sigue siendo posible para los límites de adquisición del provider; el spill de presentación propiedad de la herramienta puede retener un valor canónico adquirido completo mientras reemplaza solo la presentación. El [contrato de salida canónica de herramientas](2026-07-20-canonical-tool-output-contract.es.md) es el dueño de esa separación.

## Decisión

Un seam de almacenamiento de spill delgado más un plugin de política de spill por defecto, en un nuevo grupo `packages/spill/`:

| Paquete | Rol |
|---|---|
| `@deepseek-ai/dsh-spill` | Interfaz: `ctx.spillStore`, tipos de vocabulario, sin implementación de almacenamiento. |
| `@deepseek-ai/dsh-spill-local` | Backend local: almacenamiento de archivos privado y con ámbito de sesión en el sistema de archivos del host. |
| `@deepseek-ai/dsh-spill-policy` | Plugin de política de resultados de herramientas: envuelve los resultados de texto finales después del dispatch y reemplaza los resultados sobredimensionados con una vista previa retenida más un localizador de spill. |

No hay ningún paquete Consumer dedicado orientado al modelo. El Consumer es el pipeline de ejecución existente de `ctx.tools`: `dsh-spill-policy` consume los resultados finales de las herramientas a través del waterfall (cascada de eventos) de `tools/post-execute`, y el modelo sigue la pista de recuperación suministrada por el backend para el localizador devuelto.

### El seam de spill

El seam de almacenamiento es mínimo: guardar texto y devolver un localizador más una pista de recuperación.

```ts ignore-check
interface SpillStore {
  saveText(input: SaveTextSpill): Promise<SpillRef>
}

interface SpillSource {
  toolName: string
  callId: CallId
  label: string
}

interface SaveTextSpill {
  owner: { sessionId: SessionId }
  source: SpillSource
  suggestedName: string
  content: string
}

type SpillLocator = Branded<'SpillLocator'>

interface SpillRef {
  locator: SpillLocator
  bytes: number
  retrievalHint: string
}
```

`SpillLocator` es un [tipo marcado](../../../../packages/util/brand) (branded) orientado al modelo devuelto por el backend. El backend local lo representa como una ruta del sistema de archivos; un backend remoto o de base de datos puede representar un URI, una clave o un token de comando. Los Consumers lo tratan como opaco y lo presentan con `retrievalHint` en lugar de asumir que `read` es siempre el mecanismo de recuperación correcto. `SpillOwner.sessionId` es el espacio de nombres de almacenamiento en el momento de guardar: las sesiones bifurcadas heredan los localizadores de spill existentes del registro sembrado sin copiarlos ni re-apropiarlos, y los nuevos spills posteriores a la bifurcación usan el id de la sesión hija. Una limpieza por período de retención puede expirar los localizadores antiguos junto con otros artefactos antiguos de la sesión; el seam de spill no define una política de limpieza por sesión.

`dsh-spill-local` es dueño solo de los detalles de almacenamiento: selección del directorio con ámbito de sesión, nombres seguros, protección contra traversal de rutas, la escritura y la devolución de `{ locator, bytes, retrievalHint }`. No es dueño de la política de retención, del reemplazo de resultados de herramientas, de la búsqueda ni de la inspección de archivos. Los archivos aterrizan en `<root>/session-<hash>/<random>-<safeName>`, donde `root` es una ruta configurada o un directorio temporal privado (0700) por proceso creado de forma perezosa, el subdirectorio de sesión es un prefijo corto de `sha256(sessionId)`, y la hoja es un prefijo hexadecimal aleatorio más el `suggestedName` del llamador saneado a un único segmento de ruta (refleja el `encodeSegment` del backend JSONL). La escritura es `open(path, 'wx', 0o600)` — exclusiva y solo del propietario, de modo que un symlink plantado no puede redirigirla. El localizador es la ruta, y la pista de recuperación le dice al modelo que puede usar `read` o `grep` sobre esa ruta.

### La política de spill

`dsh-spill-policy` es un transformador de resultados de `tools/post-execute` con una única perilla de configuración:

```ts ignore-check
interface Config {
  /** Omitted means no automatic spill policy. Present means apply to oversized plain text tool results. */
  maxInlineBytes?: number
}
```

Cuando se omite `maxInlineBytes` el plugin no registra nada (un auténtico no-op). Cuando está establecido, aplica una política por defecto a los resultados finales de texto plano de las herramientas:

1. Deja que la herramienta se ejecute con normalidad, delegando mediante `next()` para que un listener posterior asiente el resultado primero.
2. Aplana el `ContentBlock[]` final aceptado solo cuando es íntegramente texto plano; un resultado con cualquier bloque que no sea texto se deja intacto.
3. Si su tamaño en bytes UTF-8 es menor o igual que `maxInlineBytes`, lo deja sin cambios.
4. Si es mayor, llama a `ctx.spillStore.saveText()` con el texto final completo.
5. Reemplaza el resultado orientado al modelo con una vista previa retenida de inicio/fin más la referencia de spill.

La vista previa es un valor por defecto de implementación propiedad de la política: una división de inicio/fin de `maxInlineBytes` mediante el `TextRetainer` de la biblioteca de retención. La configuración futura puede exponer el tamaño de la vista previa solo después de que un segundo despliegue lo necesite.

El texto de reemplazo es deliberadamente genérico porque la política solo conoce el resultado final formateado de la herramienta, no el recurso interno de la herramienta:

```text
<retained preview>

(Omitted N bytes. Full formatted result stored at: /.../session-.../....txt. Use read with offset/limit, or grep this path to search within it.)
```

Si `ctx.spillStore.saveText()` falla (permisos, ENOSPC, backend no disponible), o la llamada no tiene propietario de sesión, o no hay ningún backend cargado, el plugin registra el motivo y devuelve el resultado original sin cambios. El fallo del spill nunca convierte una llamada de herramienta exitosa en un resultado `isError` ni oculta el resultado en línea.

La política omite `read` para evitar un bucle circular `read -> archivo de spill -> read otra vez`. La configuración adicional de exclusión voluntaria queda diferida hasta que una segunda herramienta real lo necesite.

## Caso de demostración: web_fetch

`web_fetch` es el primer caso de demostración porque devuelve un resultado de texto naturalmente grande y no necesita código de spill específico de la herramienta. La herramienta es ordinaria:

```ts ignore-check
ctx.tools.register(defineTool({
  name: 'web_fetch',
  output: {
    schema: WEB_FETCH_RESULT_SCHEMA,
    render: (_args, value) => [{ type: 'text', text: formatFetchOutput(value) }],
  },
  async execute(args, exec) {
    const result = await ctx.web.fetch({ url: args.url }, exec.signal ? { signal: exec.signal } : undefined)
    return result
  },
}))
```

Con `dsh-spill-policy` configurado, un resultado de fetch formateado grande se retiene y se derrama automáticamente. Un despliegue demuestra el comportamiento estableciendo el límite de recursos del provider por encima del límite de la política:

```yaml
- id: web-fetch-http
  name: '@deepseek-ai/dsh-web-fetch-http'
  config:
    maxBodyChars: 500000

- id: spill-local
  name: '@deepseek-ai/dsh-spill-local'

- id: spill-policy
  name: '@deepseek-ai/dsh-spill-policy'
  config:
    maxInlineBytes: 50000
```

Esta separación es importante. `web-fetch-http` sigue siendo dueño de los límites de recursos (`maxResponseBytes`, `maxBodyChars`) para proteger la red, la memoria y el trabajo de decodificación. `spill-policy` es dueño solo del límite de contexto orientado al modelo, después de que el resultado ya existe. Si el provider ya devolvió `truncated: true`, el archivo de spill contiene el resultado formateado completo que la herramienta devolvió, no la página web original completa; la política no afirma lo contrario.

## Relación con la retención y el spill temprano

La retención es distinta del almacenamiento de spill:

- `@deepseek-ai/dsh-output-retention` es dueño de la mecánica de vista previa (`TextRetainer`, `ItemRetainer` y los metadatos omitidos).
- `@deepseek-ai/dsh-spill` es dueño de guardar el texto final y devolver un localizador más una pista de recuperación.
- `@deepseek-ai/dsh-spill-policy` aplica la política por defecto de resultados finales en el pipeline de herramientas, componiendo los dos.

La política de resultados finales no puede reemplazar el spill temprano propiedad de la herramienta. Parte del contenido útil no está presente en el `ToolExecutionResult.content` final:

- la salida final de `bash` ya es una cola más una ruta de spill temporal; los flujos completos de stdout/stderr viven en los archivos del executor.
- la salida final de `subagent` es la respuesta final del hijo, no el rollout del hijo.
- las herramientas futuras pueden producir artefactos de runtime que nunca están representados por su `ToolExecutionResult.content` final.

Esos casos pueden consumir `ctx.spillStore` directamente en trabajo posterior. No forman parte del primer caso de demostración.

## No objetivos

- Sin herramienta nueva orientada al modelo `artifact_read` ni `artifact_search` en v1.
- Sin configuración de retención por herramienta en v1.
- Sin argumentos de timeout/truncamiento orientados al modelo.
- Sin migración de la salida de `read` a archivos de spill.
- Sin reemplazo de los límites de recursos del provider como `web-fetch-http.maxBodyChars`.
- Sin normalización de archivos temporales de bash ni captura de rollouts de subagent en el primer corte.

## Diferido

- `saveFile()` / `linkOrCopy` para los archivos de spill existentes del executor, necesario para la normalización de bash.
- Spill propiedad de la herramienta para los rollouts de subagent (`await run.result`, leer la sesión hija en proceso antes de `run.dispose()`, guardar el JSONL).
- Exclusión voluntaria por herramienta o declaraciones de política por herramienta si la omisión integrada de `read` es insuficiente.
- Backends de almacenamiento remotos o de base de datos para entornos ACP o remotos donde una ruta local no tiene sentido.
- Política de limpieza y retención para los archivos de spill antiguos, probablemente ligada a la limpieza de sesiones.

## Pruebas

- Las pruebas unitarias de `dsh-spill` fijan el contrato del seam: registro como `ctx.spillStore`, una implementación por contexto y la liberación en la disposición.
- Las pruebas unitarias de `dsh-spill-local` cubren `saveText`, el saneamiento de `encodeSegment` (separadores/tilde/puntos de segmento completo/vacío), el directorio de hash de sesión, los permisos solo del propietario, las rutas distintas por guardado, la raíz configurada o privada y un rechazo por fallo de almacenamiento.
- Las pruebas unitarias de `dsh-spill-policy` ejecutan herramientas reales a través de `ctx.tools.execute`: no-op en modo deshabilitado, reemplazo de texto sobredimensionado, paso directo de texto pequeño/no texto, omisión de `read`, respaldo de mejor esfuerzo (fallo de guardado / sin backend / sin propietario) y composición downstream (acotar un resultado reemplazado, conservar `additionalContexts`).
- La integración de `dsh-tool-web` ejecuta `web_fetch` a través de `ctx.tools.execute` con el backend real `spill-local` más la política, demostrando que el texto orientado al modelo cambia solo por el aviso de spill deliberado mientras el archivo de spill contiene el resultado formateado completo.
- El ejemplo `tui-agent` carga `spill-local` + `spill-policy`, de modo que su smoke de Loader/PTY sin clave ejercita la ruta de carga real (la forma de exportación del plugin de espacio de nombres + `inject`).

## Consecuencias

La política por defecto solo ve el texto final formateado. No puede conservar el contenido interno del provider que ya estaba acotado ni los artefactos de runtime que nunca formaron parte del resultado. Esto es aceptable para el primer corte porque el caso de demostración es el spill de resultados finales, no el spill temprano; el spill temprano propiedad de la herramienta sigue siendo trabajo diferido.

Devolver rutas reales desde el backend local mantiene v1 simple y coincide con el comportamiento probado de las herramientas de agentes, mientras que el seam en sí solo promete un localizador opaco más una pista de recuperación, de modo que los backends remotos pueden devolver localizadores que no son archivos.

La propuesta de valor del backend local depende de que las herramientas existentes `read`/`grep` puedan inspeccionar la ruta local devuelta, incluso cuando el directorio de spill está fuera del cwd de la sesión. Eso se cumple hoy porque la política del sistema de archivos registra observaciones y protege las escrituras pero no confina las lecturas al workspace. Una política futura de confinamiento del workspace debe permitir explícitamente las rutas de spill locales o usar un backend de spill que no sea de archivos cuya pista de recuperación apunte a un lector compatible.

**Vacío de instantáneas.** Ningún escenario de instantánea de ACP cubre todavía el aviso de spill de `web_fetch` visible en el transcript (transcripción). El harness de instantáneas de ACP repite sin clave y no puede alcanzar la web en vivo, y un spill de `web_fetch` requiere un cuerpo HTTP real por encima del límite; un escenario determinista necesitaría un destino de fetch de loopback sembrado que el árbol de repetición no cablea actualmente (los ejemplos no cargan `tool-web` en absoluto). El comportamiento queda cubierto en su lugar por la prueba de integración de `dsh-tool-web` contra un servidor de loopback. Cerrar el vacío es trabajo de seguimiento: cablear `tool-web` más un destino de fetch sembrado en el ejemplo ACP y grabar después un escenario `web-fetch-spill`.

La política puede volverse demasiado grande si empieza a poseer semántica específica de herramientas. Se mantiene estrecha: solo resultados finales de texto plano. El spill temprano propiedad de la herramienta sigue siendo trabajo futuro.

## Alternativas consideradas

**Exigir que cada herramienta opte por participar con una declaración de retención.** Rechazada para v1: el objetivo es un comportamiento por defecto similar a la persistencia genérica de resultados de herramientas de Claude Code. Una única perilla de despliegue `maxInlineBytes` es suficiente para demostrar la forma.

**Convertir `tool-results` en una plataforma amplia de resultados de herramientas.** Rechazada: un nombre de paquete amplio invita a reunir en un solo seam la política de retención, el reemplazo de resultados, la redacción de las vistas previas, la búsqueda y el spill temprano. La parte compartida del almacenamiento es más pequeña: guardar texto y devolver un localizador más una pista de recuperación.

**Usar `ctx.fs.writeText` o la herramienta `write` orientada al modelo.** Rechazada: las escrituras en el sistema de archivos del workspace llevan semántica de archivos de proyecto, política de escritura/edición, estado de observación y efectos secundarios visibles al usuario. Los archivos de spill son artefactos de runtime, no ediciones del workspace autoradas por el modelo. La herramienta existente `read` puede inspeccionarlos después, pero la creación pertenece al seam de spill del runtime.

**Dejar que `web-fetch-http` descargue sin límites y confiar en spill-policy.** Rechazada: spill-policy se ejecuta después de que el resultado final de la herramienta existe y no puede proteger la red, la memoria ni los recursos de decodificación. Los límites de recursos del provider siguen siendo obligatorios.

**Fusionar la retención en el spill.** Rechazada: la retención y el spill tienen responsabilidades distintas. `TextRetainer`/`ItemRetainer` deciden qué vista previa se conserva y qué se omitió; el almacenamiento de spill solo guarda el texto final que la política le pide guardar.
