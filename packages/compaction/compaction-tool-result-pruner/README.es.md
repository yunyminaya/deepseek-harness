# @deepseek-ai/dsh-compaction-tool-result-pruner

[English](README.md) | Español

El servicio de poda sin modelo seguro para la reproducción (`ctx.toolResultPruner`). Reescribe los nodos de superficie `tool/result` por encima del presupuesto a una cabeza acotada, un marcador fijo de omisión y una cola acotada, conservando el evento original completo en el log de sesión de solo añadido.

Este es un compañero concreto de [`dsh-compaction-basic`](../compaction-basic/README.es.md), no un backend de compactación ni una herramienta orientada al modelo. Compact-basic lo lee a través del `ctx.get('toolResultPruner')` opcional, de modo que cualquiera de los dos paquetes sigue siendo componible de forma independiente.

## API del servicio

`pruneSession(session)` escanea una instantánea estable de la superficie actual. Cada resultado de herramienta por encima del presupuesto se reemplaza por un `tool/result` nuevo añadido que lleva `{ surfaceOp: { op: 'replace', start: originalSeq, end: originalSeq }, sourceEventSeqs: [originalSeq] }`. El reemplazo extiende los datos originales completos y solo cambia `content`, conservando `turn`, `step`, `callId`, los campos de error, `meta` y las adiciones de datos posteriores. El evento original sigue disponible para la persistencia, la reproducción y la inspección exacta del log.

El método lanza de forma síncrona cuando la sesión rechaza un reemplazo. Los reemplazos confirmados antes en la pasada siguen siendo durables.

`measureContent(blocks)` cuenta puntos de código Unicode en los bloques `text`. `pruneContent(blocks)` devuelve el reemplazo acotado o `null` cuando el contenido ya está dentro del umbral. Los bloques que no son de texto se conservan en sus posiciones relativas originales; el corte de texto nunca divide un par sustituto UTF-16, aunque puede dividir un clúster de grafemas de varios puntos de código.

Cada resultado emitido tiene exactamente el presupuesto de cabeza configurado, el marcador fijo y el presupuesto de cola en puntos de código de texto, no es mayor que `thresholdChars` y es estrictamente menor que la entrada que lo disparó. Por tanto, una segunda pasada no emite ningún reemplazo.

## Config

Las claves no reconocidas fallan en la construcción del plugin. La configuración resuelta está desacoplada y es profundamente inmutable.

| Clave | Obligatorio | Significado |
|---|---|---|
| `thresholdChars` | no (valor predeterminado `8192`) | Poda cuando el texto combinado supera esta cantidad de puntos de código Unicode. |
| `headChars` | no (valor predeterminado `4096`) | Puntos de código Unicode iniciales retenidos. |
| `tailChars` | no (valor predeterminado `1024`) | Puntos de código Unicode finales retenidos. |

Todos los valores son enteros; el umbral es positivo y cabeza/cola no son negativos. `headChars + marker + tailChars` debe caber dentro de `thresholdChars`, de modo que una configuración válida puede podar cada resultado por encima del presupuesto sin crecimiento ni reescrituras repetidas.

## Uso

```ts
import type { Context } from '@deepseek-ai/cordis'
import ToolResultPruner from '@deepseek-ai/dsh-compaction-tool-result-pruner'

export function apply(ctx: Context): void {
  ctx.plugin(ToolResultPruner)
}
```

## Experiencia del modelo

### Resultado de herramienta podado

#### Qué ve el modelo

Cuando un disparador de compactación califica, las solicitudes futuras ven la cabeza retenida, `\n\n[... tool result middle pruned ...]\n\n` y la cola retenida en lugar del texto eliminado. Los bloques ricos conservan su orden. El modelo no ve una segunda copia del original.

#### Efecto en tokens

Cada resultado de herramienta reescrito tiene como máximo `thresholdChars` puntos de código de texto. La poda en sí no hace ninguna llamada al modelo; compaction-basic omite el resumen cuando la solicitud remedida cae por debajo de la presión; en caso contrario, el resumidor lee la superficie podada.

#### Efecto en la caché KV

Reemplazar un resultado anterior invalida la reutilización desde el primer token cambiado. El prefijo podado es elegible para la reutilización mientras su ruta, su envelope y el historial precedente sigan siendo idénticos.

## Limitaciones conocidas y trabajo pendiente

- **Los presupuestos de caracteres no son presupuestos de tokens** — la densidad de tokens del provider varía, así que `ctx.tokenMeter` sigue siendo la autoridad para decidir si la poda alivió la presión de la solicitud.
- **La poda es sintáctica** — conserva el principio y el final sin interpretar qué líneas intermedias son semánticamente importantes.
- **Los clústeres de grafemas pueden dividirse** — el corte por puntos de código protege los pares sustitutos pero no realiza segmentación de grafemas sensible a la configuración regional.
