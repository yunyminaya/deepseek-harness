# @deepseek-ai/dsh-tool-ask-user

[English](README.md) | Español

Herramienta `ask_user_question` orientada al modelo sobre `ctx.userQuestions`. Permite al modelo hacer a la persona una pregunta concisa cuando necesita confirmación, una elección o información que le falta antes de continuar.

## Herramienta

`ask_user_question` acepta:

- `questions` — array obligatorio y no vacío de objetos de pregunta.
- `id` — id estable obligatorio en cada pregunta, repetido en la respuesta.
- `question` — texto de pregunta obligatorio para cada pregunta.
- `header` — encabezado corto opcional.
- `options` — opciones opcionales con `label` y `description`. Si recomiendas una opción, ponla la primera y añade `(Recommended)` a esa etiqueta.
- `multi_select` — si esa pregunta puede devolver más de una opción seleccionada.

La herramienta llama a `ctx.userQuestions.ask()` y devuelve el canónico `{ answers: [{ id, selected, custom? }] }`. `selected` contiene las etiquetas de las opciones; `custom` lleva una respuesta de forma libre que complementa a `selected` en una pregunta multi-select y la reemplaza en una pregunta de una sola selección. El renderizador Native conserva la forma de texto JSON compacta `{ "answers": [{ "id": "...", "selected": ["..."], "custom": "..." }] }`.

## Rol

Es el paquete Consumer del seam de user-questions. No renderiza UI y no sabe cómo se recoge la entrada; solo traduce los argumentos del modelo a `AskUserQuestionRequest` y devuelve la respuesta humana al agent loop.

## Experiencia del modelo

### Schema de la herramienta

#### Qué ve el modelo

El modelo ve el [schema generado de `ask_user_question`](../../../docs/tool-catalog.es.md#deepseek-aidsh-tool-ask-user), incluidos los ids de pregunta, los prompts, los encabezados, las opciones y las banderas multi-select.

#### Efecto de tokens

Coste de schema fijo en cada solicitud en la que la herramienta es visible.

#### Efecto en la caché KV

Prefijo estable mientras la definición y la visibilidad no cambien. El ciclo de vida del plugin o las restricciones con ámbito pueden invalidar la reutilización de este schema.

### Historial de llamada de herramienta y resultado

#### Qué ve el modelo

Las preguntas completas del modelo permanecen en los argumentos de la llamada de herramienta del asistente. Tras la respuesta humana, el siguiente paso ve JSON compacto con la forma exacta `{"answers":[{"id":"<id>","selected":["<label>"],"custom":"<text>"}]}`; `custom` se omite cuando no se usa y `selected` puede contener cero, una o varias etiquetas. La interacción de UI mientras la llamada está pendiente no es contexto de modelo.

#### Efecto de tokens

Los argumentos y el JSON de la respuesta son tokens retenidos dependientes de los datos; no hay coste de tokens mientras se espera a la persona.

#### Efecto en la caché KV

Solo añadidura; el contenido recién visible sigue el prefijo de solicitud reutilizable y no invalida las entradas existentes de la caché KV.

## Limitaciones conocidas y trabajo diferido

- **Una pregunta pendiente bloquea la llamada de herramienta hasta que la persona responda** — la herramienta no declara ningún presupuesto de `timeout-policy`; la cancelación viaja solo en el `exec.signal` del turno.
- **Los subagentes propiedad del runtime no pueden preguntar al usuario** — `ask_user_question` rechaza con `DELEGATED_CALLER` un hijo vivo propiedad de otro agent; el hijo debe incluir la pregunta o decisión sin resolver en su resultado final. El linaje duradero no decide este límite, así que una sesión con linaje reanudada como raíz de runtime puede preguntar con normalidad.
- **Las respuestas Native se renderizan como texto JSON** — el valor canónico sigue siendo estructurado, pero el resultado orientado al modelo usa JSON compacto en lugar de un vocabulario de content blocks más rico.
