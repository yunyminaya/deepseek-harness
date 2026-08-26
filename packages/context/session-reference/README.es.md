# `@deepseek-ai/dsh-session-reference`

[English](README.md) | Español

`ctx.sessionReferenceResolver` prepara instantáneas acotadas de solo lectura de otras sesiones como contexto orientado al modelo con fuente. Consume `ctx.sessionQuery` y el marcador de checkpoint de compactación independiente del backend; FTS de SQLite no es necesario. Los hosts que admiten menciones entre sesiones pueden optar por el servicio.

## API pública

- `listCandidates(agent, query?, limit?)` lista sesiones distintas de `agent.id`, filtra sin distinción de mayúsculas por id, cwd o el último título respaldado por log, y clasifica los registros del mismo cwd, luego sin cwd y luego de otro cwd, preservando el orden de creación de `listSessions()` dentro de cada grupo. Cada candidato seleccionado usa ese título como etiqueta de mención y cae al id de sesión cuando el título está ausente o es ilegible; los cuerpos de mensaje no se buscan. El método Remote unario `sessionReferenceResolver/candidates` sirve el mismo descubrimiento bajo el límite de candidatos configurado y adjunta la mención canónica de cada candidato, por lo que los consumidores de navegador llaman a `ctx.remote.sessionReferenceResolver.candidates` sin una ruta de API Proxy.
- `prepare(agent, content, references, signal?)` preserva el orden de primera mención, deduplica ids, rechaza la autorreferencia y más del límite de fuentes distintas configurado, lee cada fuente en paralelo y devuelve contenido desacoplado más cero o un `UserMessage` de contexto agregado e identificado. El servicio lo llama para menciones canónicas en mensajes directos de usuario después de que los listeners descendentes de `agent/pre-step` acepten el paso.
- `encodeSessionReferenceUri()` y `decodeSessionReferenceUri()` implementan `dsh-session:<base64url(JSON.stringify(sessionId))>` para que todo id de cadena JavaScript haga round-trip exactamente. `formatSessionReferenceMention()` emite `@[label](uri)`, y `parseSessionReferenceText()` reemplaza las menciones Markdown o las URIs canónicas desnudas con texto legible `@label` mientras devuelve las referencias estructuradas. Las menciones Markdown explícitas rechazan toda URI malformada; el texto desnudo se considera referencia solo cuando una carga con forma base64url no vacía sigue al esquema, y un candidato no canónico coincidente falla igualmente. Las menciones de esquema vacías o solo de puntuación siguen siendo texto ordinario de discusión.

## Semántica de instantánea

La preparación llama a `ctx.sessionQuery.readSurface()` una vez por fuente distinta cuando el mensaje de destino llega a `agent/pre-step`. Un mensaje encolado captura por tanto el estado de la fuente en la entrada del paso del modelo, y el contexto resultante es inmutable después de ese punto. La proyección conserva solo `user/message` directo de usuario, texto del asistente y checkpoints `user/message` que llevan el marcador de fuente canónico `dsh-compaction` de la superficie actual plegada. Los mensajes de referencia de sesión de otra fuente son contexto inyectado y se excluyen, lo que impide la propagación recursiva de instantáneas. Los eventos ensombrecidos previos a la compactación, las herramientas, el razonamiento, otros mensajes de usuario generados por plugins salvo los checkpoints de compactación marcados y los fragmentos de asistente sin terminar también se excluyen. Una fuente compactada contribuye por tanto su último checkpoint más la conversación posterior retenida, no texto ensombrecido restaurado.

La fuente de contexto es `{ kind: 'session-reference', version: 1, references }`; cada referencia registra su id de fuente y etiqueta, seq de captura, presencia de compactación, recuentos de mensajes retenidos/omitidos, bytes UTF-8 omitidos y estado de truncamiento. El listener externo de `agent/pre-step` del servicio post-procesa los mensajes directos de usuario aceptados, preserva sus ids de mensaje e inserta cada instantánea inmediatamente después del mensaje que la citó. Las ediciones de cola y el reubicación de cola a steering no necesitan manejo específico de referencias porque el análisis ocurre tras la reclamación final de la bandeja de entrada. Las menciones inválidas, las lecturas fallidas, la cancelación y los fallos de presupuesto terminan ese turno antes de que sus mensajes entren en el historial visible para el modelo. El log de destino registra el `user/message` directo legible seguido de su `user/message` de contexto con fuente; la mutación de la fuente tras la captura no puede cambiar la reproducción del destino.

## Configuración

| Clave | Por defecto | Contrato |
|---|---:|---|
| `maxReferences` | `3` | Máximo de sesiones de fuente distintas en un mensaje preparado; debe ser como máximo `3`. |
| `candidateLimit` | `50` | Recuento de candidatos por defecto devuelto a un host. |
| `maxReferenceBytes` | `65536` | Máximo de bytes JSON serializados para un objeto de referencia. |

La retención aplica `maxReferenceBytes` independientemente a cada fuente, conserva los checkpoints de compactación y el mensaje más nuevo antes de descartar unidades más antiguas que no son checkpoint, y usa el truncamiento de cabeza/cola de `dsh-output-retention` con un aviso exacto de omisión UTF-8. Si los campos serializados fijos de una fuente no caben, la preparación falla con `SESSION_REFERENCE_BUDGET_EXCEEDED` en lugar de devolver un contexto parcial.

## Model Experience

### Antecedentes de la sesión referenciada

#### Lo que ve el modelo

El modelo ve dos mensajes consecutivos de rol de usuario: el mensaje actual con su `@label` legible y luego la instantánea no confiable `## Referenced sessions`. La advertencia prohíbe seguir instrucciones, reclamaciones de permiso o peticiones de herramientas de la instantánea a menos que el usuario actual las repita explícitamente. Las etiquetas, los valores de cwd, los ids y el texto de conversación se serializan como JSON dentro de las etiquetas `<referenced-sessions>`; cada `<` de dato se emite como el escape JSON sin pérdidas `\\u003c`, por lo que el texto fuente no puede deletrear una etiqueta de enmarcado.

#### Efecto de tokens

Cada mensaje referenciado añade la advertencia fija más hasta tres instantáneas serializadas, cada una acotada independientemente por `maxReferenceBytes`. La instantánea exacta permanece en el historial de destino hasta que la compactación del destino la ensombrece o resume; los cambios en la sesión fuente no añaden más tokens.

#### Efecto de caché KV

La petición y la instantánea son mensajes de destino consecutivos de solo añadidura y preservan el historial cacheable anterior. Diferentes referencias o contenidos de captura de fuente cambian solo el nuevo sufijo; la compactación posterior del destino puede invalidar la reutilización desde su límite de reemplazo.

## Limitaciones conocidas y trabajo diferido

- **Sin descubrimiento de cuerpo** — las consultas de candidatos inspeccionan títulos plegados pero no buscan cuerpos de mensaje. Una consulta no vacía puede inspeccionar cada log de sesión persistido visible mediante el lote acotado y cancelable del servicio de consulta de sesiones; un índice de títulos dedicado puede reemplazar esa vía de descubrimiento sin cambiar los contratos de URI, instantánea o persistencia.
- **Límite de llamador confiable** — el servicio asume que su host está autorizado para leer toda sesión expuesta por `ctx.sessionQuery`; no es una herramienta de búsqueda orientada al modelo.
- **Solo proyección de texto** — los bloques de usuario y asistente no textuales no se propagan entre sesiones.
- **Sin enlace en vivo** — las referencias son instantáneas, no forks, reanudaciones, suscripciones ni mutaciones de la sesión fuente.
