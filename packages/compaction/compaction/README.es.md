# @deepseek-ai/dsh-compaction

[English](README.md) | Español

El **`CompactionEngine`** (`ctx.compaction`) define QUÉ hace la compactación — decidir cuándo el historial es demasiado grande y resumir un rango antiguo en un único nodo de superficie — sin decir CÓMO.

Este paquete posee el rol de Service Definition de la capacidad de compactación, dividido para que cada rol evolucione (e intercambie) de forma independiente:

| Paquete | Rol |
|---|---|
| `@deepseek-ai/dsh-compaction` (este) | Service Definition: servicio abstracto + eventos `compaction/*` + `CompactionResult` + constructor de fuente de checkpoint correlacionado + helpers de límite de emparejamiento de herramientas |
| `@deepseek-ai/dsh-compaction-basic` | Service Provider: presión de `ctx.tokenMeter` + retención por presupuesto de tokens + resumen con `llm.stream()` |
| `@deepseek-ai/dsh-command-compact` | Consumer: el comando humano `/compact` sobre `ctx.compaction.compactNow()` |

A diferencia del seam de bash, esta Service Definition depende de `@deepseek-ai/dsh-session` y `@deepseek-ai/dsh-llm` — los verbos del contrato se definen sobre una `Session` y su salida es el vocabulario `ContentBlock`, por lo que no se pueden expresar sin nombrar esos paquetes. Esa desviación de la guía «Service Definition depende solo de cordis» es intencional y está registrada en la [Agent Note del capability seam de compactación](../../../.agents/notes/implemented/feature/2026-06-18-compaction-capability-seam.es.md).

## API del servicio (`ctx.compaction`)

Las tres operaciones son **abstractas** — el backend posee la política de disparo, la retención, la secuenciación de eventos y el resumen. La medición reutilizable de peticiones es un servicio aparte, [`ctx.tokenMeter`](../../llm/token-meter/README.es.md), y no forma parte de esta Service Definition.

| Miembro | Semántica |
|---|---|
| `compactIfNeeded(agent, trigger, signal)` | Considera la compactación automática para `trigger: 'pressure' \| 'context-overflow'`. Un disparo por presión puede aplicar el umbral del backend y su política de cola retenida; un desbordamiento confirmado puede forzar una reducción equilibrada útil. Devuelve el `CompactionResult`, o `null` cuando no existe ningún rango seguro. La petición de resumen de un backend es una llamada directa a `ctx.llm.stream()` (no un paso del loop), por lo que la intercepción por llamada ocurre en `llm/stream`. |
| `compactNow(agent, signal)` | Compacta explícitamente un tramo antiguo equilibrado y útil incluso por debajo de la presión automática. Reserva de forma síncrona la admisión de un turno ocioso antes de ceder, no escribe nada cuando no existe ningún tramo útil, registra un intento independiente `compaction/* { turn: null }` antes del resumen y espera su checkpoint de durabilidad antes de liberar. Los fallos operativos esperables usan `ManualCompactionError`; la cancelación relanza la razón de aborto exacta. |
| `compactRegion(start, end, agent, signal?)` | Fuerza el resumen de los nodos de superficie `[start, end]` (seq inclusivos) de `agent.session` en un único nodo de reemplazo cuya fuente procede de `compactCheckpointSource(compactionId)`. **Lanza** si ya hay una compactación en curso, si `start`/`end` no son nodos de superficie, o si `start` está posicionado después de `end` en la superficie. El rango es un tramo de POSICIÓN EN SUPERFICIE, no un intervalo numérico de seq — tras un reemplazo previo que aterriza un nodo de resumen de seq alto en la posición del rango ensombrecido, el orden de superficie ya no sigue el orden de seq. |

`CompactionResult` mantiene el resumen en bruto y los seq de los eventos de bookkeeping a disposición de los llamadores junto con el rango ensombrecido y la contabilidad de tokens; su forma verificada contra deriva vive en la [referencia de estructura de datos de compactación](../../../docs/subsystems/compaction.es.md#compactionresult).

`compactIfNeeded` y `compactNow` reciben un `signal` obligatorio; el de `compactRegion` es opcional. Un backend que resume mediante `ctx.llm.stream()` **debe** reenviarlo al `GenerateOptions.signal` de la llamada, para que un aborto o una disposición de fiber desmantele el resumen en curso. Los corchetes automáticos y de región explícita recuperan su propietario numérico del turno abierto en ese momento. Los corchetes manuales no requieren turno abierto y sellan `turn: null`.

`ManualCompactionError.code` es el conjunto cerrado `busy | changed | summary | commit | persistence`. `changed` y `summary` significan que la superficie de conversación seleccionada no fue reemplazada, pero su intento fallido queda registrado en el log de sesión. `commit` es deliberadamente neutro sobre la mutación parcial, y `persistence` significa que el corchete en memoria se cerró pero su vaciado explícito falló.

## Límites de emparejamiento de herramientas

La Service Definition exporta `toolPairingBalancedBefore(session, seq)` y `toolPairingBalancedAfter(session, seq)` para ajustar y validar los bordes de compactación. Un borde seguro no tiene ninguna llamada de herramienta del asistente sin responder cruzándolo. Cada helper valida que la secuencia de eventos esté en la superficie actual y responde desde los balances almacenados en caché por corte en orden de superficie.

La caché privada por sesión está claveada por `session.surface.replaceGeneration` y el recuento de entradas de superficie procesadas. Una generación sin cambios extiende el pliegue solo con entradas de cola no vistas; una adición solo de log sin nueva entrada de superficie no hace lecturas de eventos, mientras que una generación de reemplazo reconstruye la pertenencia actual y los balances. Los seq de eventos ausentes y un `tool/result` sin una llamada abierta precedente se rechazan como estado de superficie corrupto.

## Contrato de superficie

`SurfaceEventType` es una unión cerrada — solo `user/message`, `assistant/message` y `tool/result` pueden llevar `surfaceOp`. Por tanto, un evento `compaction/*` **no puede** aparecer en la superficie. Una compactación exitosa en su lugar:

1. añade `compaction/start` (solo log) — adquiere el bloqueo,
2. resume el rango,
3. añade `compaction/summary` (solo log) con el resumen, el rango, los seq ensombrecidos, el recuento de tokens y el envoltorio de llamada provider/modelo,
4. añade un único `user/message` con `source: compactCheckpointSource(compactionId, sourceCommandId?)` y `surfaceOp: { op: 'replace', start, end }` que lleva el resumen — **la única mutación de superficie de esta operación**,
5. añade `compaction/end` (solo log) — libera el bloqueo.

La mutación de superficie (paso 4) se sitúa **dentro** del corchete de bloqueo: `compaction/end` es el último evento, por lo que el bloqueo nunca se libera antes de que aterrice la mutación. Un cuelgue entre `compaction/start` y `compaction/end` deja por tanto un bloqueo huérfano detectable (un `compaction/start` sin su `compaction/end` correspondiente), en lugar de un `compaction/end` que afirme falsamente que la compactación terminó mientras la superficie nunca se ensombreció.

El par de marcadores nombra la adquisición y liberación del bloqueo, no un contenedor de eventos exclusivo. Un `inject()` ocioso puede añadir contexto no relacionado entre un inicio y un fin manuales mientras el resumen está pendiente. La estabilidad manual por tanto revalida el tramo seleccionado en lugar de exigir igualdad de toda la superficie; el reemplazo posicional deja ese contexto inyectado visible tras el checkpoint. La compactación automática mantiene la igualdad de toda la superficie dentro de su turno activo.

`deriveMessages()` renderiza entonces el resumen como un mensaje de rol de usuario seguido de los nodos retenidos. Los eventos ensombrecidos permanecen en el log en bruto, por lo que la reproducción es determinista.

## Bloqueo

La compactación se serializa mediante un único bloqueo registrado en el log compartido por todos los puntos de entrada. La inspección de cola encuentra de forma independiente el `compaction/start` no emparejado más reciente y el `session/end-seed` más nuevo. Un inicio no emparejado después de ese límite está vivo y reporta `busy`; un inicio no emparejado más antiguo es evidencia obsoleta de un ciclo de vida de proceso anterior y no bloquea. La misma transición de end-seed limpia la traza de reproducción del compañero de invariante. Un corchete vivo no puede cruzar un `turn/start` ni un `turn/end`; durante la adopción, los límites de reparación en el prefijo heredado siguen siendo reproducibles cuando el end-seed posterior demuestra que su corchete abierto está obsoleto.

El bloqueo es el corchete duradero, no un `WeakSet`, un mutex envolvente ni un ancla del lado del cliente. `compaction/start` se añade de forma síncrona antes de que el resumen ceda. Cada fallo posterior hace exactamente un intento de `compaction/end { error }`; si ese cierre de añadido falla a su vez, el inicio no emparejado sigue siendo la señal intencional de ocupado y no se intenta ningún vaciado. Un intento manual cerrado con éxito se vacía incluso cuando reporta `changed` o `summary`, preservando el intento registrado antes de que se libere la admisión de turno.

## Eventos

Los eventos `compaction/*` extienden `SessionEventMap` (extensible por fusión) mediante fusión de declaraciones — son eventos de sesión, no `Events` de cordis, y los tres son solo log (sin `surfaceOp`). Las cargas y semánticas por evento están en el [catálogo generado de eventos del log de persistencia](../../../docs/persistence-catalog.es.md).

## Implementar un backend

Subclasea `CompactionEngine`, implementa `compactIfNeeded`, `compactNow` y `compactRegion`, y carga la subclase como plugin — se registra como `ctx.compaction`. Cada backend exitoso crea su fuente de mensaje de usuario de reemplazo con `compactCheckpointSource(compactionId, sourceCommandId?)`; el `compactionId` obligatorio correlaciona el checkpoint con su transacción `compaction/*`, mientras que `isCompactCheckpointSource()` reconoce el marcador tras la persistencia o la clonación sin depender de la identidad del backend. Una implementación basada en plantilla o en modelo puede vivir como paquete hermano sin cambiar los llamadores ni el medidor de tokens compartido.

## Reconocer un checkpoint fuera del programa host (`./checkpoint`)

`compactCheckpointSource()`, `CompactionCheckpointSource` e `isCompactCheckpointSource()` se declaran en el subpath `@deepseek-ai/dsh-compaction/checkpoint` y se reexportan desde la raíz, por lo que los consumidores del lado host siguen leyéndolos desde la raíz. El constructor requiere el `CompactionId` propietario, lo que impide que los backends escriban un marcador no correlacionado que el invariante del paquete deba rechazar. La hoja no importa cordis ni declara aumentación de módulos (la forma [`dsh-commands/brand`](../../interaction/commands/README.es.md)), que es lo que permite a un programa de cliente o de cable nombrar la fuente de checkpoint: la **raíz** del paquete no puede entrar en ese programa en absoluto, porque alcanza la raíz de `dsh-session` y esa fusión de `Context` declara el servicio `sessions` del host contra el propio del cliente (`TS2717` — un programa por lado, según [development.md](../../../docs/development.es.md#typescript-project-layout)). El adaptador de transcript del cliente web fija su literal de plugin al tipo de fuente de la hoja, por lo que renombrar allí el id del plugin es un error de compilación aquí.

## Model Experience

### Historial de conversación, cuando se invoca un backend

#### Lo que ve el modelo

Una implementación exitosa reemplaza un rango de superficie antiguo con un checkpoint de resumen de rol de usuario — un `user/message` que lleva `surfaceOp: { op: 'replace', start, end }`; los eventos en bruto siguen registrados pero dejan de aparecer en los mensajes de modelo derivados. El seam en sí no realiza ninguna reescritura.

#### Efecto de tokens

Cero tokens directos de esta Service Definition. Un backend intercambia muchos tokens de historial retenido por un resumen y deja intacta la cola reciente.

#### Efecto de caché KV

Un reemplazo de backend exitoso invalida la reutilización desde el primer token de historial ensombrecido; el seam en sí no altera una petición.

## Limitaciones conocidas y trabajo diferido

- **Comando humano, no herramienta de modelo** — `@deepseek-ai/dsh-command-compact` expone `/compact` sin argumentos a través de `ctx.commands`; no se registra ninguna herramienta de compactación orientada al modelo.
- **Parte del desbordamiento de una sola unidad queda fuera del contrato** — la compactación equilibrada por resumen no puede dividir una unidad indivisible. El compañero de poda opcional puede reparar aún un par de herramientas cerrado cuando el volumen de tool-result con texto es extraíble; un nodo grande que no es herramienta o una unidad de herramienta cuyo resto no podable es sobredimensionado no se puede compactar.
- **Un envoltorio que por sí solo se acerca a la ventana no es trabajo de compactación de superficie** — la compactación encoge el historial derivado, nunca el system prompt, las herramientas ni el prefijo de sesión.
