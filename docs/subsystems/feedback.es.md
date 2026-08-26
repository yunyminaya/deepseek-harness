# Comentarios de mensajes

[English](feedback.md) | Español

[`@deepseek-ai/dsh-message-feedback`](../../packages/feedback/message-feedback) es el encargado de los comentarios editables de los mensajes individuales del assistant. Está deliberadamente separado del evento `feedback/record` inmutable a nivel de Session: los comentarios de mensaje son un sidecar local del dominio de almacenamiento, no contenido del log de la Session ni una proyección, y no realizan ninguna entrega de telemetría.

Fuente: [`packages/feedback/message-feedback/src/types.ts`](../../packages/feedback/message-feedback/src/types.ts)

## Tipos públicos

```ts type-equiv
/** Opaque compare-and-set token for one exact feedback item revision. */
type MessageFeedbackVersion = Branded<'MessageFeedbackVersion'>
```

```ts type-equiv
/** The human's overall judgment of one assistant message. */
type MessageFeedbackRating = 'positive' | 'negative'
```

```ts type-equiv
/** One current feedback value and its opaque mutation token. */
interface MessageFeedbackItem {
  /** Stable identity of the assistant message inside the owning Session. */
  readonly messageId: MessageId
  /** Overall positive or negative judgment. */
  readonly rating: MessageFeedbackRating
  /** Optional explanation, preserved verbatim after validation. */
  readonly note?: string
  /** Equality-only token replaced by every material create or update. */
  readonly version: MessageFeedbackVersion
  /** Host-assigned creation time in Unix epoch milliseconds. */
  readonly createdAt: number
  /** Host-assigned time of the most recent material update. */
  readonly updatedAt: number
}
```

```ts type-equiv
/** Read all message feedback belonging to one persisted Session lifecycle. */
interface MessageFeedbackListRequest {
  /** Persisted Session whose sidecar should be read. */
  readonly sessionId: SessionId
}
```

```ts type-equiv
/** Current feedback values for one Session, in first-creation order. */
interface MessageFeedbackListValue {
  /** Fresh immutable item snapshots. */
  readonly items: readonly MessageFeedbackItem[]
}
```

```ts type-equiv
/** Create or replace feedback for one assistant message. */
interface MessageFeedbackPutRequest {
  /** Persisted Session that owns the target message. */
  readonly sessionId: SessionId
  /** Target assistant-message identity. */
  readonly messageId: MessageId
  /** Desired overall judgment. */
  readonly rating: MessageFeedbackRating
  /** Optional non-blank explanation. */
  readonly note?: string
  /** Observed item version, or `null` to require that no item exists. */
  readonly ifVersion: MessageFeedbackVersion | null
}
```

```ts type-equiv
/** Delete feedback for one message after observing its current version. */
interface MessageFeedbackDeleteRequest {
  /** Persisted Session that owns the sidecar. */
  readonly sessionId: SessionId
  /** Message whose feedback should be absent after this operation. */
  readonly messageId: MessageId
  /** Observed item version; ignored when the item is already absent. */
  readonly ifVersion: MessageFeedbackVersion
}
```

```ts type-equiv
/** Idempotent deletion acknowledgement. */
interface MessageFeedbackDeleteValue {
  /** Stable postcondition shared by the first deletion and every retry. */
  readonly absent: true
}
```

```ts type-equiv
/** No persisted Session header exists for the requested id. */
interface MessageFeedbackSessionNotFound {
  readonly code: 'session-not-found'
  readonly sessionId: SessionId
}
```

```ts type-equiv
/** The id does not name a derived, append-origin assistant message. */
interface MessageFeedbackTargetNotFound {
  readonly code: 'target-not-found'
  readonly sessionId: SessionId
  readonly messageId: MessageId
}
```

```ts type-equiv
/** A material mutation did not match the addressed item's current version. */
interface MessageFeedbackVersionConflict {
  readonly code: 'version-conflict'
  /** Authoritative current item, or `null` when it does not exist. */
  readonly current: MessageFeedbackItem | null
}
```

```ts type-equiv
/** A supplied note contains no non-whitespace character. */
interface MessageFeedbackNoteBlank {
  readonly code: 'note-blank'
}
```

```ts type-equiv
/** A supplied note exceeds the configured UTF-8 byte limit. */
interface MessageFeedbackNoteTooLarge {
  readonly code: 'note-too-large'
  readonly maxBytes: number
  readonly actualBytes: number
}
```

```ts type-equiv
/** Failures shared by the public message-feedback operations. */
type MessageFeedbackFailure =
  | MessageFeedbackSessionNotFound
  | MessageFeedbackTargetNotFound
  | MessageFeedbackVersionConflict
  | MessageFeedbackNoteBlank
  | MessageFeedbackNoteTooLarge
```

```ts type-equiv
/** Successful public operation result. */
interface MessageFeedbackSuccess<T> {
  readonly ok: true
  readonly value: T
}
```

```ts type-equiv
/** Rejected public operation result with a stable business failure. */
interface MessageFeedbackRejected<E extends MessageFeedbackFailure> {
  readonly ok: false
  readonly error: E
}
```

```ts type-equiv
/** Result returned by the message-feedback `list` operation. */
type MessageFeedbackListResult =
  | MessageFeedbackSuccess<MessageFeedbackListValue>
  | MessageFeedbackRejected<MessageFeedbackSessionNotFound>
```

```ts type-equiv
/** Result returned by the message-feedback `put` operation. */
type MessageFeedbackPutResult =
  | MessageFeedbackSuccess<MessageFeedbackItem>
  | MessageFeedbackRejected<
    | MessageFeedbackSessionNotFound
    | MessageFeedbackTargetNotFound
    | MessageFeedbackVersionConflict
    | MessageFeedbackNoteBlank
    | MessageFeedbackNoteTooLarge
  >
```

```ts type-equiv
/** Result returned by the message-feedback `delete` operation. */
type MessageFeedbackDeleteResult =
  | MessageFeedbackSuccess<MessageFeedbackDeleteValue>
  | MessageFeedbackRejected<MessageFeedbackSessionNotFound | MessageFeedbackVersionConflict>
```

## Datos y concurrencia

Una fila de sidecar de Session contiene su identidad de cabecera `{createdAt, cwd}` y los elementos de comentarios con clave `MessageId`. Cada elemento lleva una valoración positiva o negativa, una nota opcional, marcas de tiempo `createdAt`/`updatedAt` asignadas por el Host y su propia versión opaca. Las versiones solo se comparan por igualdad y solo contra el mensaje abordado; los llamadores no las ordenan ni las sintetizan.

`put` usa concurrencia optimista estricta: toda solicitud para un elemento existente debe coincidir con su `ifVersion` actual, incluido un no-op. Un conflicto devuelve el elemento actual autoritativo (o `null`), así que un llamador puede reconciliar una respuesta perdida o una edición concurrente sin otra lectura. Borrar un elemento ya ausente tiene éxito. Una cola por Session encierra la inspección, la lectura, la evaluación del conflicto y la escritura de la fila completa, de modo que estas garantías cubren las llamadas concurrentes en un proceso Host.

## Autoridad de destino y ciclo de vida

`SessionPersistence.inspect()` suministra la observación de la Session de destino sin publicar ni reanudar un Agent y sin confirmar reparaciones en frío. Una verificación previa en frío con `listSnapshots()` clasifica la ausencia definitiva; el fallo de inspección de una Session catalogada se propaga como fallo de infraestructura. `put` solo acepta un `assistant/message` no vacío, de origen de anexado, con el `MessageId` solicitado; los registros de origen de reemplazo, los vacíos solo de uso y los que no son de assistant no son destinos de comentarios.

La identidad almacenada `{createdAt, cwd}` debe coincidir con la cabecera inspeccionada. Un desajuste se trata como ausencia: `list` no devuelve elementos, mientras que `put` puede reemplazar la fila obsoleta por una vinculada a la identidad de cabecera actual. Los forks usan una identidad de Session nueva y no reciben copia del sidecar, incluso cuando su semilla contiene los mismos mensajes.

## Persistencia y contrato Remote

El servicio almacena filas completas de Session en el dominio de almacenamiento `message_feedback` a través de `ctx.storageDomain`. Antes de que `put` confirme una fila que referencia un mensaje de destino, un destino en vivo coincidente pasa por el punto de control canónico `ctx.sessions.flush`; tanto la ruta en vivo como la en frío se leen luego físicamente desde la secuencia cero mediante `SessionPersistence.readFrom`. La observación resultante se revalida antes de la escritura del sidecar, así que el log de destino duradero siempre precede a su confirmación de sidecar. `maxNoteBytes` es obligatorio y acota el texto de la nota por bytes UTF-8; la composición del Web Host lo fija en `8192`. El paquete publica el contrato Remote unario `messageFeedback.list`, `messageFeedback.put` y `messageFeedback.delete` del Host a través de `TypertRemoteService` y `@Remote`; la API Cordis generada a continuación es la autoridad a nivel de método.

El disposal del plugin cierra la admisión de mutaciones, drena el trabajo aceptado de la cola por Session y luego cierra el dominio de almacenamiento.

## Superficie web

[`@deepseek-ai/dsh-client-ui-message-feedback`](../../packages/client/ui-message-feedback) es el consumidor de navegador. `@deepseek-ai/dsh-api-remotes` monta la contribución `messageFeedback` generada, así que el plugin llama a `ctx.remote.messageFeedback` y nunca toca el transporte.

Los controles son la entrada `feedback` (orden 10) del slot de lista `conversation.chat.assistant-actions`, que `ui-conversation` declara y renderiza dentro de la fila IconActions del mensaje de assistant finalizado. Llegar a ese punto de renderizado exigió un cambio de fontanería: `AssistantMessageNode` ahora lleva el `messageId` opcional del evento `assistant/message`. El campo está ausente en los parciales congelados por interrupción, y el punto de renderizado omite el slot cuando está ausente. La franja se renderiza una vez por turno, en el mensaje de assistant de cierre: el Host acepta como destino todo mensaje de paso de origen de anexado, pero los pasos anteriores de un turno de varios pasos renderizan filas de herramientas en lugar de un cuerpo valorable, así que la interfaz expone un conjunto más reducido del que permite el contrato del Host.

Un `MessageFeedbackController` por Session respalda todos los controles de mensaje de esa Session: una sola lectura de `list` siembra todo el transcript, diferida al primer hover o foco en lugar de dispararse en el montaje. Cada mutación envía como `ifVersion` la versión que el controlador observó por última vez; una respuesta `version-conflict` lleva el elemento autoritativo, así que el controlador se reconcilia a partir de la respuesta en lugar de volver a buscar. Las mutaciones se serializan por Session, de modo que una operación encolada compara contra la versión confirmada. Un `connection/reset` refresca solo las Sessions ya leídas.

## Límites y limitaciones

- La cola de mutaciones es local al proceso. El dominio de almacenamiento no tiene escritura condicional entre procesos, así que varios escritores Host sobre una misma raíz de almacenamiento no tienen garantía de compare-and-swap ni de protección contra actualizaciones perdidas.
- La persistencia de sesiones no tiene API de borrado duradero. El servicio no trata `session/disposed` ni `host/session-removed` como un borrado y, por tanto, no realiza ninguna cascada ficticia; pueden quedar filas de sidecar huérfanas tras la eliminación del log fuera de banda.
- Una solicitud en el intervalo estrecho posterior al desacoplamiento en vivo pero anterior a que el catálogo de persistencia materialice la cabecera puede recibir `session-not-found`; los llamadores reintentan después de la materialización del retiro.
- Las solicitudes en frío escanean el catálogo completo de instantáneas de Session porque la persistencia no tiene una operación de metadatos de búsqueda por id. Una fila de Session tampoco tiene tope de número de elementos ni de bytes agregados; `maxNoteBytes` acota solo cada nota hasta que un consumidor concreto tenga una política de fila.
- La identidad de cabecera detecta un id reutilizado solo cuando `{createdAt, cwd}` difiere; un log clonado que conserve la misma identidad de cabecera es indistinguible según este contrato.
- El contrato del Host no registra actor autenticado ni identidad de auditoría y, por tanto, asume un límite de llamador de confianza.
- Los controles web solo aparecen en la vista de chat. Las vistas trajectory y waterfall no renderizan ninguna entrada de comentarios, aunque sus nodos de assistant lleven el mismo `messageId`.
- El sidecar no publica marcos en vivo, así que la valoración de una segunda pestaña se vuelve visible al reconectar o en la siguiente respuesta de conflicto, no de inmediato.
- El editor de notas no comprueba `maxNoteBytes` por adelantado; una nota sobredimensionada falla al guardar con `note-too-large`, no mientras se escribe.

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxmessagefeedback--messagefeedbackservice"></a>

### `ctx.messageFeedback` — `MessageFeedbackService`

Storage-domain sidecar service. It inspects persisted Session history and never creates or resumes an Agent or Session.

```ts cordis-catalog
/**
 * Read feedback belonging to the current persisted Session lifecycle.
 * A stale row from a reused Session id is invisible.
 * @param request - Session identity to inspect and list.
 * @returns current immutable items or `session-not-found`.
 */
@Remote('list') async list(request: MessageFeedbackListRequest): Promise<MessageFeedbackListResult>

/**
 * Create or replace feedback for one derived append-origin assistant
 * message. Every request must match the addressed item's current version;
 * a matching no-op returns the stored item without changing its revision.
 * @param request - target, desired value, and observed item version.
 * @returns the committed item or an explicit business failure.
 */
@Remote('put') put(request: MessageFeedbackPutRequest): Promise<MessageFeedbackPutResult>

/**
 * Delete one feedback item. Absence is successful regardless of the
 * supplied version; an existing item requires an exact version match.
 * @param request - Session, message, and observed item version.
 * @returns the stable absent postcondition, or an explicit failure.
 */
@Remote('delete') delete(request: MessageFeedbackDeleteRequest): Promise<MessageFeedbackDeleteResult>
```

Source: [`packages/feedback/message-feedback/src/index.ts`](../../packages/feedback/message-feedback/src/index.ts)
<!-- END GENERATED cordis-surface -->
