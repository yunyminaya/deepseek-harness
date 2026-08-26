# Añade un nodo de conversación del Web Client

[English](adding-a-conversation-node.md) | Español

Este tutorial añade una fila de propiedad del negocio a la vista Chat del Web Client. El plugin terminado correlaciona una familia de eventos de sesión durables en un único Context, construye State de negocio de forma incremental, publica datos de Step tipados y renderiza un Chat Node con clave sin escanear la ventana de sesión ni otros nodos renderizados. El tutorial asume que el Host ya registra los eventos y que el plugin de cliente está compuesto en el bundle de Web; las UIs externas del lado del Host y otros destinos de vista como Trajectory quedan fuera de este tutorial.

La [decisión de ensamblaje de Conversation Node](../../.agents/notes/implemented/architecture/2026-08-09-client-conversation-node-assembly.es.md) recoge el razonamiento y el modelo de motor completo. Esta guía cubre la ruta de implementación.

## 1. Diseña una familia de eventos reproducible

Elige un id de negocio estable antes de escribir la Definition. Cada evento que contribuye al mismo Node debe llevar ese id o derivarlo de forma independiente de su propio payload; el cliente nunca debe asignar una actualización al Context «el último sin terminar».

Para un trabajo de revisión, el contrato de eventos podría ser:

| Event | Rol | Datos durables requeridos |
|---|---|---|
| `review/start` | start único | `reviewId`, coordenadas de Turn/Step, título |
| `review/progress` | update | el mismo `reviewId`, coordenadas, progreso reproducible |
| `review/end` | update | el mismo `reviewId`, coordenadas, resumen final |

Usa el tipo de id branded de propiedad del productor a través del límite de proceso. Pon la fusión de `SessionEventMap` y los tipos de payload en la exportación solo de tipos del productor y, desde el paquete de cliente, importa esa exportación por sus efectos secundarios. Cada `(kind, id)` puede tener como máximo un evento start. Un negocio de un solo evento puede usar la identidad estable del evento, como `event.seq`, como su id local de la Definition.

Los eventos incrementales están soportados. Prefiere los checkpoints de valor completo cuando el productor pueda emitirlos a bajo costo, porque siguen siendo útiles cuando el start queda fuera de la ventana cargada. Cada delta debe llevar el id estable y producir State determinista al reproducirse en `seq` ascendente del log; no debe depender de memoria solo en vivo. Si la ventana de historial actual solo contiene updates, el ensamblador mantiene un Context pendiente y no construye State hasta que una página más antigua aporte el start. Si el producto debe renderizar antes de que el start esté cargado, un evento terminal o checkpoint debe llevar suficiente estado de respaldo completo para que la Definition construya ese resultado directamente; no lo recuperes escaneando eventos no relacionados.

## 2. Implementa la Definition y el payload de Chat tipado

El ejemplo mantiene las declaraciones del productor y la contribución del cliente en un solo bloque para que la relación completa sea visible. En una familia de paquetes, conserva el id branded y la declaración de `SessionEventMap` junto al productor de eventos, y deja la Definition, la fusión de datos de Chat y el renderer en el plugin de cliente.

```ts ignore-check
import { createElement } from 'react'
import type { Branded } from '@deepseek-ai/dsh-brand'
import type {
  ClientContext, ConversationLocation, ConversationNodeContext,
  ConversationNodeDefinition,
} from '@deepseek-ai/dsh-client-runtime/client'
import type { ChatNodeViewProps } from '@deepseek-ai/dsh-client-ui-conversation/client'

type ReviewId = Branded<'ReviewId'>

interface ReviewStartData {
  readonly reviewId: ReviewId
  readonly turn: number
  readonly step: number
  readonly title: string
}

interface ReviewProgressData {
  readonly reviewId: ReviewId
  readonly turn: number
  readonly step: number
  readonly completed: number
}

interface ReviewEndData {
  readonly reviewId: ReviewId
  readonly turn: number
  readonly step: number
  readonly summary: string
}

declare module '@deepseek-ai/dsh-session/types' {
  interface SessionEventMap {
    /**
     * Opens one durable review job.
     * @mode emit
     * @param data - stable identity, location, and initial display state.
     */
    'review/start': ReviewStartData
    /**
     * Records replayable progress for one review job.
     * @mode emit
     * @param data - stable identity, location, and latest progress.
     */
    'review/progress': ReviewProgressData
    /**
     * Closes one review job with its final summary.
     * @mode emit
     * @param data - stable identity, location, and final display state.
     */
    'review/end': ReviewEndData
  }
}

interface ReviewChatData {
  readonly title: string
  readonly completed: number
  readonly status: 'running' | 'completed'
  readonly summary?: string
}

declare module '@deepseek-ai/dsh-client-ui-conversation/client' {
  interface ChatNodeDataMap {
    'review-job': ReviewChatData
  }
}

declare module '@deepseek-ai/dsh-client-runtime/client' {
  interface ConversationStepDataMap {
    'review-job': ReviewChatData
  }
}

interface ReviewState extends ReviewChatData {
  readonly turn: number
  readonly step: number
}

function locationOf(context: ConversationNodeContext): ConversationLocation {
  return context.start?.location ?? context.matches[0]?.location ?? { kind: 'unresolved' }
}

function viewData(state: ReviewState): ReviewChatData {
  return {
    title: state.title,
    completed: state.completed,
    status: state.status,
    ...state.summary === undefined ? {} : { summary: state.summary },
  }
}

const reviewDefinition: ConversationNodeDefinition<ReviewState> = {
  kind: 'review-job',
  target: 'chat',
  match: (event) => {
    if (event.type === 'review/start') {
      return { id: String(event.data.reviewId), role: 'start' }
    }
    if (event.type === 'review/progress' || event.type === 'review/end') {
      return { id: String(event.data.reviewId), role: 'update' }
    }
    return null
  },
  start: (_context, match) => {
    if (match.event.type !== 'review/start') throw new Error('review-job requires review/start')
    return {
      turn: match.event.data.turn,
      step: match.event.data.step,
      title: match.event.data.title,
      completed: 0,
      status: 'running',
    }
  },
  update: (context, match) => {
    if (match.event.type === 'review/progress') {
      return { ...context.state, completed: match.event.data.completed }
    }
    if (match.event.type === 'review/end') {
      return { ...context.state, completed: 100, status: 'completed', summary: match.event.data.summary }
    }
    return context.state
  },
  publication: match => match.event.type === 'review/progress'
    ? 'animation-frame'
    : 'immediate',
  buildLocationData: (context, scope) => {
    if (scope !== 'step' || context.state === undefined) return null
    return {
      kind: 'step',
      turn: context.state.turn,
      step: context.state.step,
      key: 'review-job',
      value: viewData(context.state),
    }
  },
  buildViewNode: (context) => {
    if (context.state === undefined) return null
    return {
      key: context.key,
      kind: 'review-job',
      id: context.id,
      target: 'chat',
      anchorSeq: context.start?.event.seq ?? context.matches[0]?.event.seq ?? 0,
      location: locationOf(context),
      visibility: 'visible',
      data: viewData(context.state),
    }
  },
}

function ReviewNodeView({ node }: ChatNodeViewProps<'review-job'>) {
  const text = node.data.summary ?? `${node.data.title}: ${node.data.completed}%`
  return createElement('p', null, text)
}

export const inject = ['conversationEvents', 'slots']

export function apply(ctx: ClientContext): void {
  ctx.conversationEvents.register(reviewDefinition)
  ctx.slots.inject('conversation.chat.node', () => ctx.slots.register({
    name: 'conversation.chat.node',
    key: 'review-job',
  }, ReviewNodeView))
}
```

`match(event)` es un extractor de identidad, no un fold: solo recibe el evento actual y devuelve el id local de la Definition y el rol de ciclo de vida. Tras un match, el ensamblador localiza el Context por `(kind, id)` y llama a `start` una vez o a `update` con el State actual. Ambas funciones devuelven el State que adopta el motor; se prefiere devolver un nuevo valor inmutable, pero una función que muta y devuelve el mismo objeto tiene la misma semántica de adopción.

`buildLocationData(context, scope)` publica opcionalmente datos de propiedad de la Definition en un Turn o Step de propiedad del motor. Usa declaration merging para dar a cada clave un tipo de valor preciso. Otro Node en la misma Location puede consumir ese valor a través de su hook de slot restringido, como `useTurnData(key)`, sin recibir la sesión ni escanear `snapshot.chat.nodes`.

`target` y `buildViewNode(context)` declaran una contribución de renderizado de propiedad del target y deben aparecer juntos. Conserva `context.key` como identidad de cara a React, elige `anchorSeq` a partir de evidencia de orden durable y devuelve solo datos listos para el renderer. Una vez publicado un Node de target, sigue devolviendo la misma clave; usa `visibility: 'hidden'` cuando deba salir temporalmente del flujo visible en lugar de retirarlo con `null`.

## 3. Consulta un Context de negocio anterior solo en start

Algunas Definitions necesitan el State anterior más reciente de otro tipo de negocio. `start` recibe un `ConversationContextReader`; llama allí a `reader.previous<State>(kind)` en lugar de aceptar una colección de Contexts o escanear eventos. El reader devuelve como datos de solo lectura el Context iniciado más cercano anterior al `seq` del start actual.

El ensamblador registra esa dependencia. Si un prepend más antiguo aporta después un predecesor más cercano, cierra un hueco de ventana antes desconocido o revisa el State del predecesor, vuelve a ejecutar el Context dependiente desde `start` y reproduce sus updates en `seq` ascendente. La Definition consultada sigue siendo responsable de escribir State útil; el reader no expone métodos de consulta específicos del negocio ni otorga autoridad de mutación sobre otro Context.

## 4. Comprende las tres rutas de ingesta

El historial puede solicitarse desde la cola hacia atrás, una página cada vez, pero cada página aceptada se normaliza en `seq` ascendente antes de la reproducción del State.

| Ruta | Trabajo del motor | Comportamiento visible para la Definition |
|---|---|---|
| Replace al abrir, en resync o en reparación de huecos | Reconstruir la ventana cargada, aplicar match a cada evento una vez por Definition y luego reproducir cada Context iniciado | `start` seguido de sus updates en `seq` ascendente; los Contexts pendientes solo con updates permanecen sin State |
| Prepend de una página más antigua | Aplicar match solo a los eventos antiguos nuevos, fusionarlos en Contexts por `(kind, id)`, conservar los nodos con clave existentes y reproducir solo los Contexts y dependencias afectados | Un start recién encontrado activa sus updates acumulados; un cambio de Location o predecesor puede volver a ejecutar el Context |
| Append de un evento en vivo | Llamar una vez al `match` de cada Definition, buscar el Context correspondiente por clave y actualizar solo ese Context | Un `update` y una publicación solicitada para un evento posterior al start que coincide; sin escaneo de Contexts existentes |

Con `D` Definitions registradas, un evento entrante realiza `D` matches del evento actual y, tras un match, una búsqueda de Context por clave en tiempo constante. El código de la Definition debe conservar esa propiedad: no recorras la ventana de eventos completa, todos los Contexts, `context.matches` ni la colección de Nodes renderizados en la ruta normal de append. Usa State para los hechos acumulados, datos de Location para compartir dentro del mismo Turn/Step y `reader.previous()` para las dependencias indexadas de predecesores.

`publication` controla cuándo se materializa el State cambiado. Usa `immediate` para cambios estructurales o terminales, `animation-frame` para deltas visibles de alta frecuencia y `none` cuando el cambio de State solo alimenta una publicación posterior. El motor sigue aplicando cada update en orden de log; la cadencia solo agrupa la publicación de la vista.

## 5. Verifica la reproducción, la paginación y el renderizado

Añade pruebas enfocadas que establezcan estos resultados:

1. Una ventana completa pasada por replace produce el State final, los datos de Location, el payload de Node y el `anchorSeq` esperados.
2. Una cola solo con updates permanece pendiente; hacer prepend del start único produce el mismo resultado que un replace completo.
3. Un historial inicial seguido de append en vivo produce el mismo resultado que reproducir la ventana combinada.
4. Hacer prepend de una página más antigua añade filas anteriores sin reemplazar los valores de los Nodes con clave existentes cuyos datos no cambiaron.
5. Los deltas visibles repetidos conservan `context.key` y publican como máximo una vez por animation frame cuando se solicita.
6. El renderer con clave consume solo `node.data` y hooks de Location restringidos; no escanea la ventana de eventos de la sesión, los Contexts ni los Chat Nodes.

Usa [`packages/client/ui-conversation/src/client/conversation-nodes/assistant.ts`](../../packages/client/ui-conversation/src/client/conversation-nodes/assistant.ts) para streaming e interrupciones, [`inbox.ts`](../../packages/client/ui-conversation/src/client/conversation-nodes/inbox.ts) junto con [`message.ts`](../../packages/client/ui-conversation/src/client/conversation-nodes/message.ts) para consultas de predecesores, y [`packages/client/ui-deliverables`](../../packages/client/ui-deliverables) para una Definition que publica datos de Turn sin crear su propio Node.
