# Referencias de sesión

[English](session-reference.md) | Español

Descubrimiento de archivos respaldado por el host más solicitudes estructuradas de referencias entre sesiones y contextos de mensaje preparados. El [contrato file-reference](../../packages/context/file-reference) es el dueño de los registros de completado solo de ruta y de su gramática; el [contrato session-reference](../../packages/context/session-reference) define los URIs canónicos, la proyección de la superficie actual, el JSON seguro ante etiquetas y la retención de bytes, los errores estables y el prompt del modelo no confiable. Los adaptadores del host usan estos tipos en lugar de pasar su sintaxis de menciones de UI al núcleo del agent.

Sources: [`packages/context/file-reference/src/types.ts`](../../packages/context/file-reference/src/types.ts) · [`packages/context/session-reference/src/types.ts`](../../packages/context/session-reference/src/types.ts)

## Candidatos de archivo

`FileReferenceCandidate` es el resultado de descubrimiento solo de ruta. El agent al que se dirige aporta el alcance del directorio de trabajo; los providers deciden la clasificación y el acceso a namespaces sin leer el contenido de los archivos.

```ts type-equiv
/** One path-only completion candidate inside the target session cwd. */
interface FileReferenceCandidate {
  /** User-facing path accepted by normal prompts and filesystem tools. */
  path: string
  /** Directories keep completion open; files finish the mention. */
  kind: 'file' | 'directory'
}
```

## Entradas y candidatos

`SessionReferenceInput` es la selección independiente del host. El id es la autoridad; la etiqueta son metadatos de visualización que se transportan a la instantánea.

```ts type-equiv
/** One source session selected by a host. */
interface SessionReferenceInput {
  /** Opaque source session identity. */
  sessionId: SessionId
  /** Optional user-facing mention label. */
  label?: string
}
```

`SessionReferenceCandidate` es la salida de descubrimiento orientada al host. Su etiqueta usa el título de sesión más reciente cuando existe, mientras que el filtrado sigue buscando solo el id de sesión y el cwd, nunca el texto del transcript.

```ts type-equiv
/** One host-facing candidate from exact session metadata. */
interface SessionReferenceCandidate {
  /** Opaque source session identity. */
  sessionId: SessionId
  /** Latest log-backed title, falling back to the opaque session id. */
  label: string
  /** Source session working directory, when recorded. */
  cwd?: string
  /** Source session creation time in Unix epoch milliseconds. */
  createdAt: number
}
```

El método Remote `sessionReferenceResolver/candidates` sirve el mismo descubrimiento a los Consumer de navegador y adjunta a cada candidato su mención de prompt canónica.

```ts type-equiv
/** One discovery candidate carrying its canonical prompt mention. */
interface SessionReferenceMentionCandidate extends SessionReferenceCandidate {
  /** Canonical `@[label](dsh-session:…)` mention serialized into the prompt draft. */
  mention: string
}
```

## Mensajes preparados

La preparación conserva el contenido legible del mensaje actual y devuelve como mucho un contexto agregado.

```ts type-equiv
/** Direct message content and optional referenced-session context. */
interface PreparedReferencedMessage {
  /** Readable message content after host mention tokens are removed. */
  content: ContentBlock[]
  /** Aggregated untrusted snapshot, absent when the message has no references. */
  additionalContext?: UserMessage
}
```

## Errores

`SessionReferenceError.code` separa la configuración o entrada inválidas, la autorreferencia, los límites de recuento, el fallo de lectura de fuente, el fallo de presupuesto y la cancelación. Los protocolos del host traducen estos códigos a sus propios envoltorios de error sin inspeccionar los bytes del prompt.

```ts type-equiv
/** Stable failure codes exposed to host adapters. */
type SessionReferenceErrorCode =
  | 'SESSION_REFERENCE_INVALID_CONFIG'
  | 'SESSION_REFERENCE_INVALID_REFERENCE'
  | 'SESSION_REFERENCE_SELF_REFERENCE'
  | 'SESSION_REFERENCE_TOO_MANY'
  | 'SESSION_REFERENCE_READ_FAILED'
  | 'SESSION_REFERENCE_BUDGET_EXCEEDED'
  | 'SESSION_REFERENCE_CANCELLED'
```

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## API de Cordis

Generado desde la fuente por `scripts/gen-cordis-catalog.ts` (verificado como fresco por `pnpm run verify-cordis-catalog` en doc-sync; regenéralo con `pnpm run gen-cordis-catalog`) — los lados de idioma solo difieren en las rutas de los documentos emparejados específicas de cada localización. Los bloques de firmas usan un recinto `ts cordis-catalog` y conservan el JSDoc original de la fuente; los modos de despacho están definidos en el [primer](../cordis-primer.es.md#dispatch-modes), y la API `ctx` heredada del framework vive en [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxfilereferences--filereferenceservice-abstract-seam"></a>

### `ctx.fileReferences` — `FileReferenceService` (seam abstracto)

Capacidad del host para el descubrimiento de referencias de archivo cancelable.

```ts cordis-catalog
/**
 * List file and directory candidates for one agent's working directory.
 * @param agent - target agent whose session cwd bounds discovery.
 * @param query - path text following `@` or `@"`.
 * @param signal - caller cancellation.
 * @returns deterministic path-only candidates.
 */
abstract list( agent: Agent, query: string, signal: AbortSignal, ): Promise<FileReferenceCandidate[]>

/**
 * Remote face of {@link list}; the decorator cannot mark the abstract
 * member, so this concrete adapter carries the identical contract.
 * @param agent - target agent whose session cwd bounds discovery.
 * @param query - path text following `@` or `@"`.
 * @param signal - caller cancellation.
 * @returns deterministic path-only candidates.
 */
@Remote('list') remoteExportList( agent: Agent, query: string, signal: AbortSignal, ): Promise<FileReferenceCandidate[]>
```

Types: [Agent](core.es.md)

Source: [`packages/context/file-reference/src/index.ts`](../../packages/context/file-reference/src/index.ts)

<a id="ctxsessionreferenceresolver--sessionreferenceresolver"></a>

### `ctx.sessionReferenceResolver` — `SessionReferenceResolver`

Consumer de lectura exacta que prepara contexto de mensaje inmutable entre sesiones.

```ts cordis-catalog
/**
 * List reference candidates, ranked by working-directory affinity.
 * @param agent - target agent; self is excluded and its cwd drives ranking.
 * @param query - optional case-insensitive session-id/cwd/title substring.
 * @param limit - optional positive result cap.
 * @param signal - optional cancellation boundary for host autocomplete teardown.
 * @returns candidates labeled by latest title or, when absent, session id.
 */
async listCandidates( agent: Agent, query: string = '', limit: number = this.config.candidateLimit, signal?: AbortSignal, ): Promise<SessionReferenceCandidate[]>

/**
 * Remote face of {@link listCandidates}: the configured candidate limit
 * applies, and every candidate carries the canonical mention a host inserts
 * into the prompt draft.
 * @param agent - target agent; self is excluded and its cwd drives ranking.
 * @param query - optional case-insensitive session-id/cwd/title substring.
 * @param signal - caller cancellation.
 * @returns mention-carrying candidates in rank order.
 */
@Remote('candidates') async remoteExportCandidates( agent: Agent, query: string, signal: AbortSignal, ): Promise<SessionReferenceMentionCandidate[]>

/**
 * Snapshot all references for one accepted direct message and return one aggregated durable context.
 * @param agent - target agent; references to it are rejected.
 * @param content - already host-normalized readable message content.
 * @param references - structured source sessions in mention order.
 * @param signal - optional cancellation boundary for the active turn.
 * @returns detached content and optional referenced-session context.
 */
async prepare( agent: Agent, content: ContentBlock[], references: SessionReferenceInput[], signal?: AbortSignal, ): Promise<PreparedReferencedMessage>
```

Types: [Agent](core.es.md) · [ContentBlock](llm-streaming.es.md)

Source: [`packages/context/session-reference/src/index.ts`](../../packages/context/session-reference/src/index.ts)
<!-- END GENERATED cordis-surface -->
