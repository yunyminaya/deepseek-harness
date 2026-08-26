# dsh-output-retention

[English](README.md) | Español

Una biblioteca de **retención** ligera en dependencias: salida acotada orientada al modelo para herramientas que deben limitar cuánto contexto devuelven. Un llamador alimenta un objeto acotado con elementos o trozos de texto y obtiene el contenido retenido más los metadatos exactos de omisión.

La biblioteca es dueña **solo** de la pregunta mecánica *«¿qué conservamos y qué omitimos?»*. El código específico de cada herramienta conserva su semántica de negocio: agrupación de archivos, numeración de líneas, códigos de salida, estados de error del provider, truncado de previsualización por línea, archivos spill y la prosa orientada al modelo. Esta es la frontera que traza la [Agent Note](../../../.agents/notes/implemented/architecture/2026-07-06-tool-result-retention-library.md).

Es una **biblioteca, no un servicio ni un plugin**: sin `ctx`, no registra nada, no emite eventos. El único estado es por-retainer (una acumulación), nunca entre llamadas. Los paquetes de herramientas la importan directamente.

## API

```ts
import {
  ItemRetainer, TextRetainer,
  describeOmitted, formatRetentionNotice,
} from '@deepseek-ai/dsh-output-retention'
import type {
  Omitted, PushDecision, RetainedItems, RetainedText,
  ItemRetentionStrategy, TextRetentionStrategy, RetentionNotice,
} from '@deepseek-ai/dsh-output-retention'
```

| Export | Rol |
|---|---|
| `ItemRetainer<T>` | Acota unidades lógicas ordenadas (rutas, coincidencias de grep, fuentes). Solo `head`. `push()` → `PushDecision`; `finish()` → `RetainedItems<T>`. |
| `TextRetainer` | Acota un flujo de texto orientado a bytes. `head` / `tail` / `headTail`, con límites UTF-8 preservados en `finish()`. `push()` → `PushDecision`; `finish()` → `RetainedText`. |
| `describeOmitted(omitted, unit)` | Cláusula de omisión estandarizada (`exact` imprime un recuento; `unknown` no lo hace). |
| `formatRetentionNotice(notice, recovery)` | Une la cláusula de omisión estandarizada con la guía de recuperación propia de la herramienta. |
| `Omitted` | `none` / `exact` / `unknown` — cuánto se omitió. |
| `PushDecision` | `{ kept, truncated }` — el resultado de retención por push. |

## Modos de recurso

Los dos retainers son nombres separados, no un único recolector genérico, porque difieren en el **modelo de recurso**.

- **`ItemRetainer` acota unidades lógicas ordenadas.** Una herramienta de búsqueda puede recolectar un conjunto completo de resultados para la recuperación mediante archivo spill mientras retiene solo los primeros `maxItems` para la previsualización orientada al modelo. El recuento de omisión es exacto porque el llamador sigue alimentando cada elemento observado.
- **`TextRetainer` acota texto orientado a bytes.** `head`, `tail` y `headTail` preservan los límites UTF-8 en `finish()`; `headTail` es la forma que `dsh-spill-policy` usa para construir una previsualización acotada alrededor de un aviso de archivo spill.

## `truncated` es un hecho de presupuesto, nunca «incomplete»

`truncated` significa *que el retainer omitió contenido por lo demás disponible por culpa de un presupuesto*. No significa **en absoluto** que el upstream estuviera incompleto. Los fallos de permisos, los archivos binarios omitidos, los fallos parciales del provider, los candidatos ilegibles y el UTF-8 inválido permanecen en campos del dominio de la herramienta — nunca se pliegan en `truncated`. Confundir ambos es el bug que más invita la nomenclatura de esta biblioteca; mantenlos separados.

## Bytes, no caracteres

Los topes de texto y `omittedBytes` cuentan **bytes**, por seguridad de proceso/cuerpo (el pipe de un hijo y un cuerpo HTTP son flujos de bytes). Un trozo que atraviesa un codepoint se maneja: `finish()` recorta un codepoint parcial en cada corte para que el texto devuelto nunca introduzca un carácter de reemplazo en la frontera, y los dos lados se decodifican por separado para que un codepoint nunca se reconstruya a través del medio omitido. Los presupuestos de previsualización a nivel de carácter o de línea son una preocupación aparte, propiedad de la herramienta.

## Mapeos de herramientas

Los consumidores actuales de retención usan estos mapeos:

| Herramienta | Retainer y estrategia | Notas |
|---|---|---|
| `glob` | `ItemRetainer<FsGlobEntry>`, `head` | Recolecta la lista completa de rutas ordenadas para un archivo spill mientras retiene la primera página en línea. El mapeo de rutas, los candidatos omitidos e `incomplete` quedan fuera. |
| `grep` | `ItemRetainer<FlatGrepMatch>`, `head` | Recolecta las coincidencias para un archivo spill mientras retiene la primera página en línea. El truncado de previsualización por coincidencia, la agrupación, el ordenamiento e `incomplete` quedan fuera. |
| `bash` | `TextRetainer`, `tail` o `headTail` | El ejecutor sigue siendo dueño de los archivos spill, el estado de salida, la señal, el timeout y los trabajos en segundo plano. |
| `web_fetch` | `TextRetainer`, `head` o `headTail` | Los topes de provider/recurso siguen siendo hechos del provider; el retainer aporta solo el texto retenido y los metadatos de omisión. |
| `web_search` | `ItemRetainer<WebSearchSource>`, `head` | Estandariza el aviso de «fuentes limitadas» cuando los providers devuelven más fuentes de las que el resultado orientado al modelo debería incluir. |

`read` permanece fuera de esta biblioteca genérica. Su helper `read-render` es dueño de un contrato de paginación específico de archivos — `offset`/`limit`, números de línea, `totalLines`, errores de offset fuera de rango, truncado de previsualización por línea y un tope de bytes sobre la ventana seleccionada — que es un renderizador de ventana de líneas. Un único recuento `Omitted` no puede representar ambos lados de esa ventana.

## Forma de uso

```ts ignore-check
// glob: keep the first page inline while still collecting the full list for spill.
const retainer = new ItemRetainer<FsGlobEntry>({ kind: 'head', maxItems: globMaxResults })
const allEntries: FsGlobEntry[] = []
for await (const entry of candidates) {
  allEntries.push(entry)
  retainer.push(entry)
}
const { items, truncated, omitted } = retainer.finish()

// bash: keep a head + tail, read to process exit.
const out = new TextRetainer({ kind: 'headTail', headBytes: headCap, tailBytes: tailCap })
child.stdout.on('data', (chunk: Buffer) => { out.push(chunk) })
const { text, omittedBytes } = out.finish()

// A footer: the library standardizes the omission clause; the tool owns recovery words.
const footer = formatRetentionNotice(
  { scope: 'grep', strategy: 'head', unit: 'items', limit: grepMaxMatches, kept: items.length, omitted },
  ({ kept }) => `Results capped at ${kept}. Narrow the pattern, path, or include to see more.`,
)
```

## Experiencia del modelo

Indirectamente, a través de los consumidores de herramientas que renderizan el contenido retenido y los metadatos de omisión.

#### Efecto de KV Cache

Sin invalidación directa; el consumidor nombrado es dueño de cualquier cambio de prefijo de solicitud.

## Limitaciones conocidas y trabajo diferido

- **La retención de elementos admite solo `head`** — las semánticas de tail, head/tail, paginación, agrupación y completitud del provider siguen siendo de la herramienta.
- **La retención de texto está orientada a bytes** — las ventanas de líneas y caracteres, como la paginación de `read`, requieren un renderizador aparte, y un corte puede descartar bytes parciales de frontera UTF-8 para mantener válido el texto devuelto.
