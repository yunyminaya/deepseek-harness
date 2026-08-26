# @deepseek-ai/dsh-token-meter

[English](README.md) | Español

Medición de tokens consciente del replay a través del servicio singleton `ctx.tokenMeter`. Avanza un único fold aislado por sesión a partir del registro duradero, de modo que la compactación y otros plugins sensibles a la presión puedan compartir la contabilidad sin depender de `CompactionEngine`.

## Configuración

El estimador no tiene ajustes. Usa a propósito una única heurística fija: cuatro caracteres por token más el overhead estructural de roles, bloques y campos del envelope de petición. Cualquier clave se rechaza; la capacidad de modelo pertenece al adaptador que posee una ruta exacta de provider/modelo y está disponible a través de `ctx.llm.resolveModelInfo().context`.

## Contrato de medición

`ctx.tokenMeter` expone directamente dos operaciones:

- `measure(session, requestHeader?)` devuelve la presión de la petición y la superficie valorada actual en una revisión del registro consumido.
- `estimateMessage(message)` valora un mensaje con la heurística fija.

`measure()` sincroniza una vez y devuelve una única instantánea separada y profundamente inmutable. `totalTokens` es la presión de petición y respuesta, mientras que `surfaceTokens` es el total heurístico solo de superficie y equivale a la suma de `nodes[].tokens`. Una sobrescritura de `requestHeader` afecta solo a los campos de presión; los campos de superficie siguen describiendo la sesión actual. Cada llamada clona los nodos posicionales, así que la medición es O(superficie).

El fold rastrea instantáneas completas de cabecera de petición, límites de paso, añadidos y reemplazos de superficie, mensajes de asistente exitosos, uso del provider y los seq de chunk citados por cada mensaje de asistente. El uso del provider se reutiliza solo cuando el envelope canónico de petición de la última llamada exitosa coincide con el envelope medido y su total no es inferior al ancla heurística completa de esa llamada; un éxito posterior reemplaza el ancla anterior. En caso contrario se estiman el envelope y la superficie actuales completos. Los cambios de superficie se mantienen con signo relativo a un ancla coincidente, incluidos los deltas negativos tras reemplazos que encogen.

La contabilidad de uso suma cubos disjuntos de entrada, lectura de caché, escritura de caché y salida; el razonamiento no se suma otra vez. Cada llamada exitosa registra un ancla de asistente, incluidas las llamadas sin contenido. Una lista `sourceEventSeqs` vacía explícita significa un stream de provider conocido como vacío, mientras que una lista legacy ausente trata conservadoramente la salida de asistente duradera como salida del provider.

## Proyecciones de sesión

Cuando la composición aporta `ctx.sessionProjections`, token-meter registra tres unidades a través de una fiber hija opcional.

`tokenUsage` transporta los `uncachedInputTokens`, `outputTokens`, `cacheReadTokens` y `cacheWriteTokens` del registro duradero completo. Los chunks de uso se cuentan incluso cuando una petición falla después; un uso final de mensaje de asistente para el mismo `(turn, step)` reemplaza esa muestra en lugar de contarla dos veces. El razonamiento sigue siendo una subdivisión de la salida. El único slot de última muestra se apoya en una propiedad de ordenación del registro de sesión: una vez que un paso posterior informa de uso, un registro legal nunca vuelve a informar de uso para un paso anterior.

`contextPressure` transporta `pressureTokens` opcional — el tamaño de prompt más reciente informado por el provider, que suma la entrada sin caché más las lecturas y escrituras de caché —, `projectedTokens` opcional y `contextWindow` opcional del registro `request/context` más reciente. Ambas cifras permanecen ausentes hasta que un provider informa de uso; la capacidad permanece ausente para una ruta cuyo adaptador no anuncia ninguna. La salida queda excluida, así que `pressureTokens` se mantiene quieto mientras un turno transmite y avanza cuando la siguiente petición informa de su uso.

`projectedTokens` es lo que costaría el prompt de la SIGUIENTE petición: la muestra más el revalorado heurístico de todo lo que la superficie ganó o perdió desde que se tomó, acotado a cero y pasado por el mismo `surface-fold.ts` que reproduce el servicio de medición. Solo se estima el delta, así que la cifra se mantiene anclada al provider mientras reacciona en el momento en que aterriza contenido — o una compactación ensombrece un tramo. Ese último caso es la razón de que el campo exista: la compactación resume mediante una llamada directa a `ctx.llm.stream()` y no añade uso propio, así que `pressureTokens` solo informa del prompt previo a la compactación hasta que completa un turno entero más. Las pantallas de ocupación leen `projectedTokens`.

`contextBreakdown` transporta `systemTokens`, `toolsTokens` y `messageTokens` heurísticos — la composición del contexto, no su tamaño facturado por el provider. Las cifras del envelope se revaloran con último-gana en cada `request/header`; la cifra de mensajes reproduce `surface-fold.ts` — el mismo fold posicional que ejecuta `measure()` — así que equivale a `measure().surfaceTokens` en cada límite de evento y la compactación la encoge igual que encoge la siguiente petición. Las tres cifras usan la heurística fija del servicio de medición y son estimaciones: no sumarán `projectedTokens`, cuyo ancla de provider transporta exactamente el error — el texto CJK y los schemas JSON se infravaloran mucho a cuatro caracteres por token — que las filas de composición aún contienen. Preséntalas como una composición aproximada, nunca como un total.

Las tres unidades usan la línea base de proyección estándar, el frame vivo, el almacén de mayor-seq-gana y las rutas de checkpoint JSON. Descargar token-meter elimina las tres claves. Una composición sin el seam de proyección conserva el comportamiento existente del servicio de medición.

### La ocupación de contexto es una aproximación, por diseño

Los campos de ocupación son registros de último-gana independientes y **no** son una observación atómica de una sola petición. Cambiar de modelo empareja la capacidad nueva con la muestra de la ruta anterior hasta que la siguiente petición informa de uso, y `pressureTokens` describe la última petición, no la superficie tal como está ahora mismo — `projectedTokens` lleva esa muestra hacia delante sobre el movimiento de la superficie, pero su ancla sigue siendo la petición más antigua.

Esto es deliberado. Un porcentaje de ocupación es una cifra de referencia orientada al usuario, no un registro de facturación ni una entrada de compuerta — nada en el harness toma decisiones a partir de ella, y la compactación lee `measure()` en su lugar. Una UI calcula la ocupación dividiendo la presión medida por la capacidad resuelta por separado para el modelo seleccionado.

El [Agent Note](../../../.agents/notes/implemented/architecture/2026-07-29-projected-token-usage-and-request-context.es.md) registra la comparación atómica de pares rechazada. Los Consumers que necesiten una cifra exacta en el mismo límite deben llamar a `measure()` en su propio límite de petición en lugar de leer esta proyección.

## Composición

```yaml
- name: '@deepseek-ai/dsh-token-meter'
- name: '@deepseek-ai/dsh-compaction-basic'
```

Ambos plugins tienen valores por defecto utilizables. El medidor permanece independiente del enrutado de modelos y de la compactación opcional. Un despliegue configura la capacidad en su adaptador de LLM y la política de compactación en `dsh-compaction-basic`.

## Experiencia de modelo

Indirectamente, a través de Consumers como `dsh-compaction-basic`; el propio servicio no añade prompt, mensaje, schema, herramienta ni llamada de modelo.

#### Efecto de KV Cache

Sin invalidación directa; el Consumer nombrado es dueño de cualquier cambio de prefijo de petición.

## Limitaciones conocidas y trabajo diferido

- **La heurística fija es aproximada** — el contenido sin uso de provider reutilizable se valora por recuento de caracteres más overhead estructural, no con un tokenizador exacto del provider ni un serializador de peticiones.
- **Cada medición clona la superficie actual** — las instantáneas inmutables coherentes hacen las lecturas O(superficie), incluidas las comprobaciones de presión por debajo del umbral.
- **El uso del provider solo es reutilizable para un envelope canónico idéntico** — los cambios de prompt, prefijo, herramientas, provider, modelo o configuración de llamada caen deliberadamente en la estimación heurística completa.
- **Los seq de fuente legacy ausentes se manejan conservadoramente** — los mensajes de asistente sin `sourceEventSeqs` no pueden distinguir la salida del provider de las reescrituras de listeners, así que el fold evita reclamar un stream de chunks conocido como vacío o exacto.
