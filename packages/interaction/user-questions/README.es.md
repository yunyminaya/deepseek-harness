# @deepseek-ai/dsh-user-questions

[English](README.md) | Español

Service Definition de interacción con el usuario. Es dueño de `ctx.userQuestions`, el servicio que una herramienta orientada al modelo o un plugin de permiso usa cuando necesita pausar el trabajo y pedir una decisión a la persona.

## Servicio: `UserQuestionService` (clave de ctx: `userQuestions`)

### API pública

- `ctx.userQuestions.registerProvider(provider): () => void` Registra el provider del lado de la UI. Solo puede haber un provider activo en un contexto; la disposición lo desregistra.
- `ctx.userQuestions.ask(request): Promise<AskUserQuestionAnswer>` Pregunta al provider activo y espera la respuesta.

### Tipos clave

- `AskUserQuestionRequest` — `{ questions: [{ id, question, detail?, header?, options?, multiSelect?, intent? }], agent?, signal? }`; `detail` aporta texto de apoyo que los providers renderizan junto con la pregunta sin convertirlo en una etiqueta de opción. Cuando está presente, `agent` debe ser la raíz de runtime viva exacta del registro.
- `AskUserQuestionOption` — `{ label, description? }`.
- `AskUserQuestionIntent` — `{ kind: 'plan-review', approve }`; la intención de presentación etiquetada que se describe abajo.
- `AskUserQuestionAnswer` — `{ answers: [{ id, selected, custom? }] }`.
- `UserQuestionProvider` — implementación de UI con `ask(request)`.
- `UserQuestionError` — subclase de `HarnessError` con códigos como `EMPTY_QUESTIONS`, `BAD_INTENT`, `NO_PROVIDER`, `DUPLICATE_PROVIDER`, `ASK_ABORTED`, `CALLER_NOT_LIVE` y `DELEGATED_CALLER`.

Para una pregunta de una sola selección, `custom` reemplaza la opción seleccionada y `selected` queda vacío. Para una pregunta multi-select, `custom` puede complementar las etiquetas de `selected`. Una UI puede conservar un elemento omitido como `{ id, selected: [] }`, manteniendo la forma de respuesta existente y reteniendo las demás respuestas del lote.

Cuando una solicitud lleva un agent, `ask()` autentica su identidad exacta a través del `AgentRegistry` vivo y admite solo una raíz de runtime. El linaje duradero no es autoridad: una sesión con profundidad histórica de delegación puede preguntar después de reanudarse como una raíz de runtime nueva, mientras que un hijo vivo propiedad de otro agent se rechaza aunque su profundidad duradera sea cero. Las solicitudes programáticas sin agent conservan la vía de provider existente.

### Intención de presentación

`intent` declara que una pregunta ES un tipo de decisión conocido, de modo que una UI que reconoce la etiqueta puede presentarla como tal — `plan-review` dice que `detail` es un plan en revisión, y `dsh-plan-mode` lo fija en la pregunta `exit_plan_mode`. Una intención cambia solo la presentación: una UI que la respeta responde con las mismas etiquetas de opción que enviaría una UI genérica, y una UI que no conoce la etiqueta renderiza la lista de opciones genérica, de modo que los llamadores leen los mismos campos de respuesta en ambos casos. `approve` nombra la etiqueta que aprueba en lugar de depender del orden de las opciones. `ask()` rechaza con `BAD_INTENT` las dos afirmaciones que ningún tipo puede expresar: un `approve` que no nombra ninguna de las opciones propias de esa pregunta, y una intención sobre una pregunta sin `detail` — la cosa de la que se declara revisión.

## Rol

Es el paquete de Service Definition. Consumers como `@deepseek-ai/dsh-tool-ask-user` dependen de este servicio; el runtime del host web aporta el Service Provider incluido. El loop permanece sin cambios: una llamada de herramienta espera una promesa, y el resultado de la herramienta reanuda el agent loop normal.

## Experiencia del modelo

Indirectamente, a través de `dsh-tool-ask-user`, que retiene una respuesta de provider correcta como JSON compacto o uno de estos fallos: `Error: ask_user_question was aborted before the user answered`, `Error: ask_user_question requires at least one question`, `Error: human interaction requires the exact live calling agent when an agent is supplied`, `Error: human interaction is unavailable while the calling agent is owned by another live agent; include the unresolved question or decision in the child agent's final result`, `Error: no user-questions provider is registered` o `Error: <message>`. Esperar a la persona no añade tokens.

#### Efecto en la caché KV

Sin invalidación directa; el consumer nombrado es dueño de cualquier cambio en el prefijo de solicitud.

## Limitaciones conocidas y trabajo diferido

- **Un provider por contexto** — no hay enrutamiento ni fan-out hacia varias UIs; un segundo registro lanza `DUPLICATE_PROVIDER` y, sin ninguno registrado, `ask()` lanza `NO_PROVIDER` en lugar de degradarse.
- **El vocabulario es solo la forma de formulario de pregunta** — opciones seleccionables más texto custom opcional; las formas de interacción más ricas (selectores de archivo, confirmaciones de vista previa de diff) aún no tienen vocabulario de seam.
