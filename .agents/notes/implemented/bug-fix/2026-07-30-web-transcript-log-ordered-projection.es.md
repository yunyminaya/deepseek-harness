# Agent Note: La conversación del browser es una transcripción humana en orden de log

Status: implemented

English | [中文](2026-07-30-web-transcript-log-ordered-projection.zh.md)

## Problema

El cliente browser construía su conversación desde la superficie modelo-visible: `FoldAdapter` corría el `SurfaceManager` del core sobre la ventana de historial y leía `surface.nodes`. Una compactación exitosa reemplaza un rango de superficie con un nodo checkpoint, así que en el momento en que ese reemplazo aterrizaba el flujo web colapsaba cada mensaje que sombreaba en una única fila de contexto atenuada — conversación que el usuario ya había leído. No se perdió nada del log; el defecto era enteramente de la proyección, y [el terminal y el gateway del host se arreglaron igual](2026-07-29-human-transcript-append-origin.es.md) mientras el browser quedó para este cambio.

El orden de superficie hacía dos problemas más estructurales. No es seq-ascendente tras un reemplazo — `SurfaceManager` empalma el checkpoint de seq alto en la posición del rango que sombrea — así que los nodos solo-log que se fusionaban en ese array por seq numérico (filas de slash-command, nodos congelados interrumpidos) podían flushearse por delante del checkpoint y no volver a intercalarse en la cola retenida. Y como la paginación ya no gasta cuota `maxMessages` en copias de reemplazo, una página puede llevar un checkpoint cuyo `surfaceOp.start` queda fuera de la ventana; el fold del core rechaza ese rango, así que `nodes()` caía a un escaneo lineal permisivo tras un `console.error` y publicaba un flag `foldDegraded` describiendo el fallo.

## Decisión

`TranscriptAdapter` reemplaza a `FoldAdapter` y jamás consulta el orden de superficie. Proyecta la ventana cruda en orden de log: cada evento de superficie de origen-append (`isAppendSurfaceEvent`) en su propia posición de log, más un marcador `CompactionSummaryNode` por checkpoint de compactación aterrizado. Una compactación aterrizada conserva por tanto la conversación que sombreó del lado del modelo, y el marcador reporta dónde dejó el modelo de ver ese historial en vez de borrarlo. Las copias de reemplazo solo-modelo quedan fuera de la transcripción: un `tool/result` podado y un `assistant/message` regenerado reescriben un nodo para el modelo y no marcan frontera en la conversación. Todo lo que debe enviar exactamente lo que el modelo ve sigue leyendo la superficie; esta es la proyección humana, y ambas quedan ahora separadas en los dos frontends.

El orden de nodos es seq-monotónico por construcción, y tres cosas siguen. El par solo-log `command/run` / `command/done` se pliega en `CommandNode`s que se empalman en un array ya-monotónico por seq — sin anclas, sin reordenar. `Session` conserva la propiedad de los nodos congelados interrumpidos y los fusiona por sus seqs fraccionales con un sort simple, que ahora es exactamente el orden de flujo. Y una ventana cuyo checkpoint cita un rango sombreado fuera de ella no tiene rango que resolver, así que el marcador renderiza y nada se registra.

`foldDegraded` desaparece de `ConversationSnapshot`, y con él los centinelas de relleno, la aritmética `baseSeq` que necesitaban y `degradedSeqs()`. Existían solo para satisfacer la aserción `seq === index` del fold del core y sobrevivir a su throw; el fold que describen ya no corre. Borrar el flag es parte del fix y no limpieza posterior — `degradedSeqs()` ya era casi la proyección en orden de log, alcanzada tras un error lanzado en vez de intencionada.

El texto de resumen del marcador, el conteo de ítems reemplazados y el conteo estimado de tokens sombreados vienen del evento `compaction/summary` citado por el checkpoint, jamás del payload del checkpoint encuadrado, que es un sobre de instrucciones escrito para el modelo. Un corte de ventana que deja ese evento fuera vuelve esos campos indisponibles, el mismo soft-fall que un resultado de herramienta sin llamada, y una página posterior que aporte el evento los resuelve.

El [comando de compactación manual](../feature/2026-07-30-queued-manual-compaction.es.md) devuelve el seq del evento de resumen como `CommandResult.sourceEventSeq` exitoso, y `command/done` persiste esa referencia opcional. El chat empareja solo un comando `/compact` con nombre exitoso cuya referencia iguala exactamente un `CompactionSummaryNode.summaryEventSeq` cargado. El comando en curso renderiza primero `compact · Compacting context…`; tras aterrizar el checkpoint, la misma React key renderiza un disclosure `compact` colapsado en la posición de flujo del checkpoint con el conteo y la estimación de tokens. El rechazo de input, la falta de historial compactable, la cancelación y el fallo permanecen como filas genéricas de comando con texto completo del handler. La compactación automática no tiene referencia de comando y conserva el marcador independiente de contexto-compactado.

La referencia explícita de evento importa porque la compactación manual permite inyección duradera de contexto mientras su resumen asíncrono corre: las filas de comando y checkpoint no están garantizadas como adyacentes. El evento de ciclo de vida del comando gana un campo opcional, pero la transacción de compactación, el sobre RPC y la superficie modelo-visible no cambian; los logs persistidos pre-release sin el campo conservan el soft-fall anterior de dos filas y no exigen migración.

## Reconocer un checkpoint: una declaración, fijada en compilación

El reconocimiento necesita las tres condiciones, como en el terminal: `event.type === 'user/message'`, la fuente del plugin checkpoint del seam de compactación, **y** `isReplacementSurfaceEvent(event)`. Un `user/message` con fuente de plugin que *añade* es contexto inyectado — una tarjeta de referencia de sesión — y no una compactación.

Lo inalcanzable desde un programa `packages/client/*` es la **raíz** de `dsh-compaction`, no el paquete. La raíz alcanza la raíz de `dsh-session`, cuya fusión cordis `Context` declara el `sessions: SessionStore` del host contra el `sessions: ISessions` del cliente — `TS2717`, la regla de un-programa-por-lado en [development.md](../../../../docs/development.md#typescript-project-layout) — y eso aplica también a una import type-only, porque la colisión es un hecho del compilador y no del bundler.

La respuesta del repo para exactamente esto es un subpath hoja sin cordis, y este cambio añade uno: `COMPACT_CHECKPOINT_SOURCE` e `isCompactCheckpointSource` viven ahora en `packages/compaction/compaction/src/checkpoint.ts`, que no importa cordis ni augmenta módulo (la forma `dsh-commands/brand` / `dsh-llm/message`), y la raíz re-exporta ambos para que todo consumidor lado-host — los helpers de chat del terminal, la proyección de `dsh-session-reference` — quede sin cambio. El adaptador fija su literal a esa declaración con una import type-only:

```ts
import type { CompactionCheckpointSource } from '@deepseek-ai/dsh-compaction/checkpoint'
const COMPACT_PLUGIN: CompactionCheckpointSource['plugin'] = 'compact'
```

Renombrar el id de plugin del Service Definition es ahora un error de compilación en el cliente: `TS2322: Type '"compact"' is not assignable to type '"compaction"'`. La importación debe permanecer **type-only** — una importación de valor de cualquier paquete `@deepseek-ai` que no sea ni platform module ni capa wire inline-safe la rechaza la puerta de pureza del client (`packages/client/tsdown.client.ts`), cuyo propio mensaje registra que las importaciones type-only se borran y jamás la alcanzan. Una importación hoja type-only necesita tanto una entrada `paths` en `tsconfig.base.json` como `{"path": "../../compaction/compaction"}` en los `references` de `packages/client/runtime/tsconfig.json`: las reglas de `rootDir` composite aplican también a importaciones borradas, y sin la referencia el diagnóstico es `TS6059`/`TS6307`.

`packages/client/ui-conversation/tests/conversation-node-definitions.client.spec.ts` es la mitad conductual, conduciendo la Definition de compactación con registros de checkpoint y procedencia y probando que una página más antigua puede llenar datos de resumen ausentes. La importación hoja type-only de la Definition mantiene al cliente aislado de la raíz del paquete compact y de las fusiones `Context` lado-host alcanzables a través de ella.

La divergencia con el terminal es por tanto estrecha: ambos frontends reconocen un checkpoint desde la misma declaración — el terminal importa por valor `isCompactCheckpointSource` lado-host, donde ninguna puerta aplica, y el cliente fija el tipo.

## Para qué eran las anclas posicionales de #835, y por qué se disuelven en vez de perderse

La rama no fusionada de cola-de-compactación-manual arregla el mismo bug de intercalado registrando un ancla por evento — la cola de superficie al tiempo de append — y re-dirigiendo las anclas sombreadas al checkpoint. Ese mecanismo existe para que las anclas posicionales sobrevivan al **reordenamiento** de superficie. La transcripción humana jamás se reordena, así que las anclas no tienen qué re-dirigir: el pre-requisito se elimina, no se descarta el fix. El mecanismo no existe en este codebase.

## Alternativas consideradas

**Importar por valor el predicado** desde la hoja nueva y añadir `dsh-compaction` a la allowlist `INLINE_SAFE` del client. Descartado: el cliente necesita el id de plugin, no el predicado — un tipo basta, y una importación borrada jamás alcanza la puerta de pureza, así que nada tiene que admitirse en ella. La allowlist solo importaría para una importación de valor, y allí es un mal trade: `INLINE_SAFE` coincide por *prefijo* de specifier, así que admitir el paquete admite su raíz importadora de cordis junto con la hoja.

**Una regla de forma desnuda** — todo `user/message` de reemplazo es una compactación. Descartado: correcto hoy solo porque la compactación es el único productor de `user/message` de reemplazo, sin nada que lo atrape si eso cambia. La spec de fijación cuesta un archivo y elimina exactamente ese riesgo.

**Taguear el checkpoint lado-host** a través de la proyección o el contrato wire. Descartado: lo más alineado con la regla «colaborar a través de servicios cordis», pero el cliente hoy pliega `SessionEvent`s crudos, así que significa un cambio de contrato wire desproporcionado para un predicado puro.

**Mover la propiedad de nodos congelados al adaptador** (`nodes(extraNodes)`), como hace la rama no fusionada. Descartado: los nodos interrumpidos vienen del barrido `turn/end` que `Session` ya corre sobre la ventana, y con una transcripción seq-monotónica la forma simple es correcta — el adaptador devuelve nodos, la sesión fusiona los congelados por seq. Ensanchar la firma del adaptador no compraría nada y partiría el barrido de su producto.

**Conservar `foldDegraded` como flag defensivo.** Descartado: describía un fallo específico de un fold que ya no corre. Un flag sobre el que ningún consumidor puede actuar, alcanzable solo a través de un `console.error`, es un contrato falso.

**Emparejar la fila `/compact` más cercana con el siguiente checkpoint.** Descartado: la inyección de contexto puede aterrizar entre ambos, y los registros de ciclo de vida concurrentes o malformados deben degradar sin robar otro checkpoint. El resultado del comando nombra en cambio el evento de resumen autoritativo, y las referencias ambiguas no emparejan nada.

**Parsear el texto de asentamiento en inglés para los conteos de ítems y tokens.** Descartado: el copy del handler es texto de presentación, no un contrato de datos estable. El marcador lee el payload estructurado `compaction/summary` que ya posee ambos valores.

## Consecuencias

La compactación ya no borra el historial web; una sesión compactada varias veces muestra un marcador por compactación aterrizada, en orden de log, y la misma ventana renderiza idéntica en vivo y tras un resume en frío. El agujero de paginación queda cerrado por construcción en vez de defendido, y `ConversationSnapshot` pierde un campo publicado, lo que tocó trece archivos.

`ConversationNode` gana un octavo brazo, así que cada consumidor exhaustivo creció un caso: `MessageItem` renderiza el marcador a través del nuevo `CompactionItem`, y el layout de trayectoria ensancha su brazo sin-celda para que un marcador no aporte celda pero sí avance el cursor de duración.

El contrato de rendimiento queda sin cambio y ahora es más simple de enunciar: un append materializa un nodo, un evento que no cambia ningún nodo conserva la referencia del array previo — así que una tormenta de chunks no cuesta nada y `nodes()` ni se recalcula — y los nodos sin cambio conservan su identidad de objeto. La ventana sigue creciendo con la longitud de la sesión y no con la superficie, que es el trade que el fix existe para hacer; una compactación solía acotar la proyección para exactamente las sesiones largas a las que la compactación sirve.

El escenario e2e web ahora siembra un ciclo de vida real de comando manual alrededor de una transacción de compactación sobre su turno grabado, así que el golden aria fija el comportamiento completo a través del host real y un browser real: el prompt grabado y la salida completa de herramienta siguen en pantalla, exactamente una fila `compact` reporta escala tras ellos, y su disclosure abre el resumen exacto. La grabación semilla queda intacta y sigue siendo modelo-auténtica — el replay deriva la compactación manual de la superficie de la propia grabación.

## Diferido

La [decisión archivada de progreso de compactación](../../archived/feature/2026-07-30-compaction-progress-visibility.md) del terminal usa el bracket independiente en vivo para conducir un indicador de una celda y no cambia esta proyección del browser.
