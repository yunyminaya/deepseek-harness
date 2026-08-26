# Interacción con el usuario

[English](user-questions.md) | Español

El seam user-questions de [dsh-user-questions](../../packages/interaction/user-questions). Es el vocabulario neutral respecto al provider que usa un plugin de herramientas o de permisos cuando necesita que la persona responda antes de que el agent (agente) pueda continuar. Las superficies de UI proporcionan el `UserQuestionProvider` activo; el runtime del host retransmite las solicitudes a su cliente conectado.

Fuente: [`packages/interaction/user-questions/src/index.ts`](../../packages/interaction/user-questions/src/index.ts)

## Opciones de pregunta

`AskUserQuestionOption` contiene una opción seleccionable. `label` es el texto de la opción visible para la persona usuaria y también el valor seleccionado que ve el modelo; `description` es texto de ayuda de UI opcional.

```ts type-equiv
/** One selectable answer offered to the user. */
interface AskUserQuestionOption {
  /** User-facing label. */
  label: string
  /** Optional extra context rendered by capable UIs. */
  description?: string
}
```

## Intención de presentación

`AskUserQuestionIntent` declara opcionalmente un tipo de decisión conocido. Se etiqueta en `kind` para poder añadir más intents; una UI que no reconoce una etiqueta muestra la lista de opciones genérica. Un intent solo cambia la presentación: una UI que lo respeta responde con las mismas etiquetas de opción que enviaría una UI genérica, así que quien llama lee los mismos campos de respuesta en ambos casos. `approve` nombra la opción afirmativa en lugar de fiarse del orden de las opciones. `ask()` rechaza las dos afirmaciones que ningún tipo puede representar: un `approve` que no nombra ninguna opción de su propia pregunta, y un intent en una pregunta sin `detail`.

```ts type-equiv
/**
 * A caller-declared presentation intent: the question IS this kind of
 * decision, so a UI that recognises the tag may present it as such instead of as a
 * generic option list. Tagged so further intents can be added; a UI that does
 * not know a tag renders the generic flow, and the answer encoding is identical
 * either way — an intent changes presentation only, never the protocol.
 */
type AskUserQuestionIntent = {
  /** A plan submitted for review: `detail` is the plan markdown `ask()` requires, and the decision approves or declines it. */
  kind: 'plan-review'
  /**
   * The option label that approves the plan; every other option declines it.
   * Named rather than positional so no UI infers the verdict from option order.
   * An `approve` naming no option of its own question is rejected at `ask()`.
   */
  approve: string
}
```

## Elemento de pregunta

`AskUserQuestionItem` es una pregunta dentro de una solicitud. Quien llama aporta un `id` estable, que se devuelve junto con la respuesta para que las preguntas en lote sigan siendo enrutables. El `detail` opcional transporta texto de apoyo que los providers muestran con la pregunta pero mantienen fuera de las etiquetas de opción seleccionables.

```ts type-equiv
/** One question in a user-questions request. */
interface AskUserQuestionItem {
  /** Stable caller-provided question id, echoed in the answer. */
  id: string
  /** The question to display. */
  question: string
  /** Optional supporting detail rendered with the question but kept out of option labels. */
  detail?: string
  /** Optional short heading/group label. */
  header?: string
  /** Optional choices the UI can render as a menu. */
  options?: AskUserQuestionOption[]
  /** Whether more than one option may be selected. Defaults to single-select. */
  multiSelect?: boolean
  /** Optional presentation intent for capable UIs; absent asks for the generic option list. */
  intent?: AskUserQuestionIntent
}
```

## Solicitud ask

`AskUserQuestionRequest` es la solicitud entre paquetes. `questions` es un array para que una UI pueda presentar preguntas relacionadas en un mismo flujo conservando un id estable por respuesta. Cuando está presente, `agent` es exactamente la parte que llama en vivo; el seam de interacción solo lo admite mientras el registro en vivo identifica esa instancia como raíz del runtime.

```ts type-equiv
/** Request for a human answer. */
interface AskUserQuestionRequest {
  /** Questions to display. */
  questions: AskUserQuestionItem[]
  /** Exact live calling agent, when the request came from an agent tool call. */
  agent?: Agent
  /** Abort signal for the owning tool/step. */
  signal?: AbortSignal
}
```

## Respuesta

Los providers devuelven un elemento de respuesta por id de pregunta. `selected` contiene las etiquetas de opción seleccionadas, y `custom` transporta una respuesta libre de «Otro» cuando la persona escribió una. En una pregunta de selección única, `custom` anula la opción seleccionada y `selected` queda vacío. En una pregunta de selección múltiple, `custom` puede complementar las etiquetas de `selected`. Una UI también puede usar un elemento con `selected` vacío y sin `custom` para conservar una pregunta saltada en un lote por lo demás completado.

```ts type-equiv
/** Answer to one question. */
interface AskUserQuestionAnswerItem {
  /** The answered question id. */
  id: string
  /** Selected option labels. May accompany custom text for a multi-select question. */
  selected: string[]
  /** Optional free-text "Other" answer. */
  custom?: string
}
```

```ts type-equiv
/** The human's answer. */
interface AskUserQuestionAnswer {
  /** Structured answers keyed by question id. */
  answers: AskUserQuestionAnswerItem[]
}
```

## Provider

Solo puede haber un provider activo por contexto. El registro del provider está ligado al effect, de modo que HMR/disposal retira la UI activa.

```ts type-equiv
/** UI-side provider for user questions. */
interface UserQuestionProvider {
  ask(request: AskUserQuestionRequest): Promise<AskUserQuestionAnswer>
}
```

## Errores

`UserQuestionError` extiende `HarnessError`, así que `ctx.tools.execute()` conserva `{ name, code }` para los fallos de herramienta visibles para el modelo, como `EMPTY_QUESTIONS`, `NO_PROVIDER`, `ASK_ABORTED` o la cancelación desde la UI.

```ts type-equiv
/** Stable error taxonomy for user-questions failures. */
class UserQuestionError extends HarnessError {
  constructor(message: string, code: string, options?: ErrorOptions) {
    super(message, code, options)
    this.name = 'UserQuestionError'
  }
}
```

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.md).

<a id="ctxuserquestions--userquestionservice"></a>

### `ctx.userQuestions` — `UserQuestionService`

`ctx.userQuestions`: one active UI provider plus an `ask()` API.

```ts cordis-catalog
/**
 * Register the UI provider. Only one provider may be active in a context.
 *
 * @param provider UI-side implementation that collects answers.
 * @returns Disposer that unregisters this provider.
 */
registerProvider(provider: UserQuestionProvider): () => void

/**
 * Ask the active UI provider and wait for the user's answer.
 *
 * When a caller supplies an agent, human interaction is valid only for the
 * exact live runtime root. Runtime ownership, not durable session lineage,
 * decides this boundary: an owned child has no human answerer and would
 * block forever, while a lineage-bearing session resumed as a new runtime
 * root may ask normally.
 *
 * @param request Questions, owner agent, and abort signal.
 * @returns The answer chosen or typed by the human.
 * @throws {UserQuestionError} code `CALLER_NOT_LIVE` when a supplied
 *   agent is not the registry's exact live instance, or `DELEGATED_CALLER`
 *   when that live agent is owned by another agent.
 */
async ask(request: AskUserQuestionRequest): Promise<AskUserQuestionAnswer>
```

Source: [`packages/interaction/user-questions/src/index.ts`](../../packages/interaction/user-questions/src/index.ts)
<!-- END GENERATED cordis-surface -->
