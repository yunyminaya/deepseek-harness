# Agent Note: Biblioteca de retención de resultados de herramientas

Status: implemented

[English](2026-07-06-tool-result-retention-library.md) | [中文](2026-07-06-tool-result-retention-library.zh.md) | Español

## Problema

Varias herramientas visibles para el modelo ya acotaban la cantidad de contexto que devuelven, pero cada una es dueña de un mecanismo y un vocabulario locales distintos: `bash` conserva una cola (tail) más archivos de spill, la búsqueda web limita las listas de fuentes, la captura web limita el cuerpo del contenido, y el descubrimiento con `glob` / `grep` necesita una primera página en línea mientras conserva los metadatos exactos de omisión para el conjunto completo de resultados. Un único helper `truncate(text)` no puede cubrir esos casos: las herramientas de elementos necesitan recuentos de elementos y agrupación fuera de la primitiva, mientras que las herramientas de texto necesitan presupuestos de bytes y cortes de cabeza/cola seguros para UTF-8.

La abstracción compartida que necesitan las herramientas es la **retención (retention)**, no una colección genérica. Un llamador introduce elementos o trozos de texto en un objeto acotado y más tarde recibe el contenido retenido más los metadatos exactos de omisión. El código específico de cada herramienta sigue siendo dueño de la semántica de negocio: agrupación de archivos, numeración de líneas, códigos de salida, estados de error del provider, archivos de spill y prosa visible para el modelo. La biblioteca común es dueña solo de la pregunta mecánica «¿qué conservamos y qué omitimos?».

## Decisión

`@deepseek-ai/dsh-output-retention` vive en `packages/util/` (a la par de `dsh-brand` y `dsh-timeout`) y es dueña de la salida acotada visible para el modelo. Es una biblioteca de clases y funciones puras, **no** un servicio ni un plugin de Cordis: no recibe `ctx`, no registra nada, no conserva estado entre llamadas y no emite eventos. Los paquetes de herramientas la importan directamente cuando necesitan salida acotada.

La biblioteca tiene dos retainers (retenedores) independientes:

- `ItemRetainer<T>` maneja unidades lógicas ordenadas, como rutas, coincidencias de grep o fuentes de búsqueda. En v1 solo admite retención `head` (cabeza), dejando la forma del retainer abierta a estrategias de retención adicionales más adelante.
- `TextRetainer` maneja flujos de texto orientados a bytes, como stdout/stderr de bash o cuerpos de respuesta web. Admite retención `head`, `tail` (cola) y `headTail` (cabeza y cola) preservando los límites UTF-8 en `finish()`.

Ambos retainers devuelven un pequeño `PushDecision` tras cada `push()` para que los llamadores sepan si esa unidad/trozo se retuvo por completo y si el resultado acumulado ya está truncado. Los recuentos de omisión son exactos porque los llamadores siguen introduciendo cada elemento/trozo observado.

```ts ignore-check
/**
 * How much content the retainer omitted.
 *
 * `unknown` is reserved for callers that omit without a count; the retainers
 * themselves return `none` or `exact`.
 */
type Omitted =
  | { kind: 'none' }
  | { kind: 'exact'; count: number }
  | { kind: 'unknown' }

interface PushDecision {
  kept: boolean
  truncated: boolean
}

/**
 * Final result for ordered logical units.
 */
interface RetainedItems<T> {
  items: T[]
  truncated: boolean
  seen: number
  kept: number
  omitted: Omitted
}

/**
 * Final result for text streams.
 *
 * The returned `text` is safe to send to a formatter; the retainer does not add
 * tool-specific headers, exit markers, XML tags, or recovery instructions.
 */
interface RetainedText {
  text: string
  truncated: boolean
  omittedBytes: Omitted
}
```

### Estrategias

La retención de elementos admite una ventana head. La retención de texto admite ventanas de bytes head, tail y headTail.

```ts ignore-check
type ItemRetentionStrategy =
  | {
      /** Keep the first `maxItems` units. Use for `glob`, `grep`, and web sources. */
      kind: 'head'
      maxItems: number
    }

type TextRetentionStrategy =
  | {
      /** Keep the first `maxBytes` bytes. */
      kind: 'head'
      maxBytes: number
    }
  | {
      /** Keep the final `maxBytes` bytes. Requires reading to the end. */
      kind: 'tail'
      maxBytes: number
    }
  | {
      /** Keep a stable prefix and suffix, omitting the middle. Requires reading to the end. */
      kind: 'headTail'
      headBytes: number
      tailBytes: number
    }
```

### Mapeo de herramientas

`read` queda deliberadamente fuera de la biblioteca de retención v1. Su helper `read-render` es dueño de un contrato de paginación específico de archivos: `offset` / `limit`, números de línea, `totalLines`, errores de rango fuera de límites, truncamiento de vista previa por línea y un tope de bytes sobre la salida seleccionada que puede detener el escaneo a mitad de ventana. Eso es un renderizador de ventana de líneas, no una primitiva de retención genérica. Puede compartir futuros helpers de avisos neutros, pero no debería pasar su ventana ya seleccionada por `ItemRetainer`.

`FsGlobEntry` y `FlatGrepMatch` a continuación son las formas de elementos previstas para las herramientas de descubrimiento, no exportaciones existentes de la biblioteca de retención. `FsGlobEntry` es una ruta derivada del backend, y `FlatGrepMatch` es una coincidencia de grep sin agrupar antes de que el backend agrupe las coincidencias retenidas por archivo.

`glob` usa `ItemRetainer<FsGlobEntry>` con `{ kind: 'head', maxItems: globMaxResults }` tras recopilar la lista completa de rutas ordenadas. La herramienta conserva la primera página retenida en línea y puede guardar la lista completa a través del seam de spill. El mapeo de rutas, los candidatos omitidos y `incomplete` quedan fuera del retainer.

`grep` usa `ItemRetainer<FlatGrepMatch>` con `{ kind: 'head', maxItems: grepMaxMatches }` antes de agrupar. El ejecutor analiza la salida de ripgrep, mapea las rutas, aplica el truncamiento de vista previa por línea e introduce las coincidencias planas. Tras `finish()`, la herramienta agrupa las coincidencias retenidas por archivo y puede guardar la lista completa de coincidencias a través del seam de spill cuando el resultado en línea está limitado. La agrupación no forma parte del retainer porque el tope es el total de coincidencias, no de archivos; el truncamiento de vista previa por coincidencia e `incomplete` también quedan separados de la retención a nivel de resultado.

`bash` puede usar `TextRetainer` con `tail` o `headTail` y leer hasta que el proceso termine. El ejecutor de bash sigue siendo dueño de los archivos de spill, el estado de salida, la señal, el timeout y el comportamiento de los trabajos en segundo plano; el helper de retención solo sustituye la contabilidad ad hoc de cabeza/cola en memoria cuando ese comportamiento se desea. La propiedad de los trabajos de larga duración sigue siendo ortogonal al [runtime genérico de herramientas de larga duración](2026-06-20-generic-long-running-tool-runtime.es.md).

`web_fetch` puede usar `TextRetainer` con `head` o `headTail`, o conservar los topes de cuerpo propiedad del provider cuando el provider debe leer y decodificar internamente. En cualquier caso, el `truncated` del resultado de la captura sigue siendo un hecho del provider/herramienta, y la biblioteca solo suministra el texto retenido y los metadatos de omisión.

`web_search` puede usar `ItemRetainer<WebSearchSource>` con `head`. Los providers actuales suelen devolver un array, así que esto es a posteriori, pero igualmente estandariza los avisos.

### Avisos

La biblioteca expone una forma de aviso neutra y un hook de formateo mínimo, pero las herramientas aportan las palabras visibles para el usuario. Un pie de grep dice «Reduce el patrón, la ruta o el include»; un pie de captura web dice «Captura una URL o sección más específica»; bash puede señalar un archivo de spill. El retainer no puede conocer esas acciones de recuperación.

```ts ignore-check
interface RetentionNotice {
  scope: string
  strategy: 'head' | 'tail' | 'headTail'
  unit: 'items' | 'bytes' | 'chars' | 'lines'
  limit: number | { head: number; tail: number }
  kept: number
  omitted: Omitted
}

const formatGrepNotice = (notice: RetentionNotice): string =>
  formatRetentionNotice(
    notice,
    ({ kept }) => `Results capped at ${kept}. Narrow the pattern, path, or include to see more.`,
  )
```

El hook de formateo es deliberadamente pequeño: una herramienta convierte un `RetentionNotice` en su propio texto de pie. El helper puede estandarizar la redacción de la omisión, pero no es dueño de la orientación de recuperación.

`truncated` significa que el retainer omitió contenido por lo demás disponible a causa de un presupuesto. No significa que el upstream estuviera incompleto. Las herramientas conservan campos separados para los fallos de permiso, los archivos binarios omitidos, los fallos parciales del provider, los candidatos ilegibles, el UTF-8 inválido y cualquier otra condición de «no se pudo inspeccionar».

## Consecuencias

**Lo que se entregó.** `@deepseek-ai/dsh-output-retention` exporta `ItemRetainer`, `TextRetainer`, los tipos de resultado (`RetainedItems`, `RetainedText`), los tipos de estrategia (`ItemRetentionStrategy`, `TextRetentionStrategy`), `Omitted`, `PushDecision`, `RetentionNotice` y los helpers de avisos neutros `describeOmitted` / `formatRetentionNotice` — sin dependencia de Cordis ni de ningún paquete de herramientas. Las pruebas unitarias cubren la retención head de elementos con recuentos exactos de omisión, la retención head de texto, la retención tail de texto, la retención de bytes cabeza-cola, presupuestos de cero, el manejo de límites UTF-8 (codepoints de 2, 3 y 4 bytes y bytes de inicio inválidos en cada corte) y la redacción de omisión desconocida.

**Lo que está documentado pero aún no migrado.** `glob`, `grep`, `bash`, `web_fetch` y `web_search` tienen sus mapeos documentados en el [README del paquete](../../../../packages/util/output-retention/README.es.md), pero no todas las herramientas se han migrado a la biblioteca en este cambio; la migración es deliberadamente un trabajo de seguimiento aparte. `read` está documentado como fuera de alcance a propósito: su contrato de ventana de líneas `read-render` (`offset`/`limit`, `totalLines`, errores de rango, truncamiento de vista previa por línea, un tope de bytes sobre la ventana seleccionada) no es retención genérica, y un único recuento `Omitted` no puede representar ambos lados de una ventana de líneas.

**Los límites que la biblioteca sostiene.** `truncated` significa que el retainer omitió contenido por lo demás disponible a causa de un presupuesto; nunca significa que el upstream estuviera incompleto. Los estados específicos de las herramientas — `incomplete`, fallos de permiso, fallos parciales del provider, omisiones de binarios, recuperación de la ruta de spill de bash, UTF-8 inválido — permanecen en campos del dominio de la herramienta, fuera del retainer. Cuando un cambio futuro migre una herramienta, el README y las pruebas de ese paquete deben demostrar que el texto del resultado visible para el modelo no cambia salvo por la redacción deliberada de los avisos.

**Compromisos aceptados.** La API v1 admite deliberadamente solo retención `head` de elementos y `head` / `tail` / `headTail` de texto; las ventanas, los presupuestos agrupados, los topes sensibles al orden y el control de parada del upstream esperan a que un segundo consumidor demuestre la necesidad. La retención de texto cuenta bytes por seguridad del proceso/cuerpo, dejando los presupuestos de vista previa a nivel de carácter y de línea como preocupaciones separadas propiedad de la herramienta.

## Alternativas consideradas

**Solo `truncate(text)` a posteriori.** Rechazada: coincide con el caso de uso de truncamiento de historial/salida de herramientas de Codex, pero pierde los recuentos de elementos, los límites de agrupación, las ventanas de bytes seguras para UTF-8 y los metadatos exactos de omisión.

**Un `Collector<T>` genérico con callbacks conectables.** Rechazada para v1: oculta los dos modos de recurso importantes. La retención de elementos lógicos cuenta elementos; la retención de texto cuenta bytes y preserva los límites UTF-8. Los nombres separados `ItemRetainer` y `TextRetainer` hacen explícita esa diferencia manteniendo pequeña la API.

**Poner la ventana de `read` detrás de `ItemRetainer`.** Rechazada para v1: `read` es el único consumidor de ventanas actual, y su semántica es paginación de archivos, no retención genérica. Un único recuento `Omitted` no puede representar ambos lados de una ventana de líneas, y `read` también carga `totalLines`, errores de rango fuera de límites, truncamiento de vista previa por línea y un tope de bytes sobre la salida seleccionada. Mantener `read-render` propiedad de la herramienta evita hacer crecer la biblioteca compartida alrededor de un caso especial.

**Convertir el truncamiento en parte de `ToolExecutionResult`.** Rechazada: el registro de herramientas tendría que entender la orientación de recuperación, la agrupación, la numeración de líneas, el estado de salida y la semántica del provider específicos de cada herramienta. La retención es una biblioteca usada por el renderizador Native de una herramienta; la proyección visible para el modelo sigue siendo propiedad de la herramienta mientras el [valor canónico](2026-07-20-canonical-tool-output-contract.es.md) puede retener el resultado adquirido completo.

**Exponer límites en cada schema de herramienta visible para el modelo.** Rechazada como opción por defecto: el grep de Claude Code expone `head_limit` / `offset`, pero este harness mantiene los presupuestos rutinarios como configuración de despliegue salvo que el modelo necesite genuinamente control de paginación. Un futuro campo de continuación tipo read puede añadirse por herramienta; no pertenece a la primitiva de retención compartida.
