# @deepseek-ai/dsh-message-feedback

[English](README.md) | Español

Feedback editable propiedad del Host para un mensaje de asistente finalizado. El paquete registra `ctx.messageFeedback`, persiste una fila de sidecar ligada al ciclo de vida por Sesión en el dominio de almacenamiento y publica el contrato Remote unario del Host `messageFeedback.list`, `messageFeedback.put` y `messageFeedback.delete`. Es independiente del evento inmutable a nivel de sesión `feedback/record` y no realiza ninguna transferencia de telemetría. La [Agent Note del sidecar de message-feedback](../../../.agents/notes/implemented/architecture/2026-08-10-message-feedback-sidecar.es.md) es dueña del límite de diseño.

Los tipos públicos de petición, valor, versión y fallo se exportan desde la raíz del paquete y desde `@deepseek-ai/dsh-message-feedback/types`; [`src/types.ts`](src/types.ts) es su fuente.

## Configuración

| clave | significado |
|---|---|
| `maxNoteBytes` | Entero seguro positivo obligatorio: longitud máxima en bytes UTF-8 de una nota opcional. |

Las notas deben contener al menos un carácter que no sea espacio en blanco, pero el texto aceptado se almacena verbatim en lugar de recortarse. Omitir `note` significa que el valor deseado no tiene nota, así que un `put` material con versión coincidente borra una nota existente. La validación de la nota precede a la búsqueda de la Sesión y por tanto puede devolver `note-blank` o `note-too-large` para una Sesión ausente sin tocar la persistencia.

```yaml
- id: message-feedback
  name: '@deepseek-ai/dsh-message-feedback'
  config:
    maxNoteBytes: 8192
```

El servicio inyecta `storageDomain`, `sessionPersistence` y `sessions`. Su dominio durable es `message_feedback`, con una fila de tabla `sessions` por `SessionId`.

## Datos, ciclo de vida y durabilidad

`MessageFeedbackItem` contiene `messageId`, `rating: 'positive' | 'negative'`, una `note` opcional, un `version` opaco solo de igualdad y marcas de tiempo Unix en milisegundos `createdAt`/`updatedAt` asignadas por el Host. Una actualización material preserva `createdAt`, reemplaza `version` y evita que `updatedAt` retroceda. `list` devuelve instantáneas inmutables nuevas en orden de primera creación; actualizar un elemento conserva su lugar, mientras que borrarlo y recrearlo después añade un elemento nuevo.

Cada fila almacenada lleva la identidad de cabecera de la Sesión inspeccionada `{createdAt, cwd}`. Una discrepancia se trata como ausencia: `list` devuelve un `items` vacío, `delete` devuelve la postcondición de ausencia y `put` puede reemplazar la fila obsoleta por una ligada a la identidad actual. Esto cerca un `SessionId` reutilizado cuando su identidad de cabecera difiere. Las bifurcaciones usan una identidad de Sesión distinta y no reciben copia de la fila de feedback.

`SessionPersistence.inspect()` proporciona una observación segura en frío sin publicar ni reanudar un Agent y sin comprometer una reparación en frío. Para una Sesión sin propietario vivo, `listSnapshots()` decide primero la ausencia definitiva; un fallo de `inspect()` para una Sesión catalogada sigue siendo un fallo de infraestructura en lugar de adivinarse como `session-not-found`. `put` acepta solo un `assistant/message` no vacío de origen de adición con el `MessageId` solicitado; los mensajes de origen de reemplazo, los registros de asistente vacíos solo de uso y los registros que no son de asistente devuelven `target-not-found`.

Después de la validación inicial, `put` establece una barrera de durabilidad antes de escribir el sidecar. Una Sesión viva coincidente se confirma a través del checkpoint canónico `ctx.sessions.flush`, y después tanto la ruta viva como la fría se leen físicamente desde la secuencia cero a través de `SessionPersistence.readFrom`. La identidad de cabecera y el objetivo de la observación resultante se validan de nuevo. Un participante de flush ausente, una identidad cambiada, un objetivo desaparecido o un fallo de lectura física impiden la confirmación del sidecar, así que el feedback durable nunca precede al mensaje objetivo durable.

El feedback de mensaje no es contenido del registro de la Sesión ni una proyección de la Sesión. No emite ningún evento `feedback/record`, no entra en el historial del modelo y no desencadena la liberación de telemetría `FEEDBACK_ONLY`.

## Contrato de servicio y Remote del Host

Los mismos tres métodos de `MessageFeedbackService` son publicados por `TypertRemoteService` y `@Remote`; los nombres de los endpoints del Host son `messageFeedback.list`, `messageFeedback.put` y `messageFeedback.delete`. Cada método devuelve una unión de negocio discriminada: `{ ok: true, value }` o `{ ok: false, error }`. Los fallos operativos de almacenamiento, corrupción o falta de oyente de durabilidad se rechazan en lugar de etiquetarse erróneamente como errores de negocio.

| Método | Petición | `value` de éxito | `error.code` rechazado |
|---|---|---|---|
| `list` | `MessageFeedbackListRequest { sessionId }` | `MessageFeedbackListValue { items }` | `session-not-found` |
| `put` | `MessageFeedbackPutRequest { sessionId, messageId, rating, note?, ifVersion }` | `MessageFeedbackItem` confirmado | `session-not-found`, `target-not-found`, `version-conflict`, `note-blank`, `note-too-large` |
| `delete` | `MessageFeedbackDeleteRequest { sessionId, messageId, ifVersion }` | `MessageFeedbackDeleteValue { absent: true }` | `session-not-found`, `version-conflict` |

`MessageFeedbackVersionConflict` devuelve el elemento `current` de autoridad, o `null` cuando no existe ningún elemento. Esto permite al llamador conciliar la valoración, la nota y la versión actuales sin una segunda petición `list`. `MessageFeedbackNoteTooLarge` devuelve tanto `maxBytes` como `actualBytes`. El agregado Remote del Client todavía no monta la contribución de cliente generada; los llamadores del Host pueden usar el contrato de servicio/Remote sin ese ensamblaje de cliente.

## Compare-and-set e idempotencia

`ifVersion: null` solicita solo la creación; cada petición para un elemento existente exige su versión actual exacta, incluida una no-op cuyo valor deseado ya coincide. La comprobación es por mensaje y no por Sesión, así que cambiar un elemento no entra en conflicto con otro. Cada creación o actualización material asigna un token UUID opaco nuevo, lo que impide que las escrituras obsoletas crucen un ciclo de valor ABA.

Una no-op con versión coincidente devuelve el elemento ya almacenado con versión y marcas de tiempo sin cambios. Después de una respuesta de éxito perdida, un reintento con el token antiguo recibe `version-conflict.current`; el llamador puede comparar ese elemento de autoridad con su valor deseado sin una lectura adicional. `delete` ignora `ifVersion` cuando el elemento ya está ausente y siempre devuelve la postcondición estable `{ absent: true }` después del éxito.

Una cola de promesas por Sesión encierra la inspección, la validación de durabilidad, la lectura del sidecar, la comparación y la escritura de la fila completa. Estas semánticas serializan las mutaciones concurrentes a través de una instancia de servicio; el propio dominio de almacenamiento no tiene escritura condicional entre procesos.

La disposición del plugin cierra la admisión de mutaciones, drena todas las operaciones ya aceptadas en las colas por Sesión y solo entonces cierra el dominio de almacenamiento. Una mutación enviada después de la disposición se rechaza como fallo de ciclo de vida en lugar de entrar en un dominio en cierre.

## Model Experience

### Estado local de message-feedback

#### Lo que ve el modelo

Nada. `ctx.messageFeedback` no registra ninguna herramienta, sección de prompt, contexto orientado al modelo ni evento de Sesión; el feedback permanece en un sidecar propiedad del Host salvo que un Consumer documentado por separado lo exponga explícitamente.

#### Efecto de tokens

Cero. Ninguna petición, resultado, valoración, nota, marca de tiempo o fallo de este paquete entra en una petición del modelo.

#### Efecto de KV Cache

Independiente. Listar o mutar el feedback de mensaje no toca un prefijo de petición del modelo y no puede invalidar una entrada de caché de provider por lo demás reutilizable.

## Limitaciones conocidas y trabajo diferido

- **Faltan el agregado de Client y la interfaz de usuario** — el contrato Remote del Host se distribuye, pero la contribución del agregado Remote del Client y cualquier Consumer de UI se poseen por separado y quedan diferidos.
- **El compare-and-set es de un solo proceso** — la cola por Sesión serializa solo una instancia de servicio; varios procesos de Host escribiendo en una raíz de almacenamiento aún pueden perder actualizaciones porque el dominio de almacenamiento no expone ninguna escritura condicional entre procesos.
- **Sin cascada durable de borrado de Sesión** — la persistencia de la Sesión no tiene API de borrado, y `session/disposed`/`host/session-removed` significan desmontaje en lugar de borrado durable. El servicio por tanto conserva filas vacías y puede dejar filas huérfanas después de la eliminación del registro fuera de banda en lugar de borrar el feedback válido al desmontar.
- **Ventana de retiro de desmontaje/catálogo** — una petición en el estrecho intervalo después del desmontaje en vivo pero antes de que el catálogo de persistencia materialice la cabecera puede recibir `session-not-found`; los llamadores reintentan después de la materialización del retiro.
- **La identidad de cabecera no es una huella del contenido** — `{createdAt, cwd}` detecta la reutilización solo cuando esos campos difieren; un registro clonado que conserva la misma identidad de cabecera es indistinguible.
- **Límite de llamadores de confianza** — `list`/`put`/`delete` no llevan ningún actor autenticado ni identidad de auditoría. Un despliegue debe exponer el gateway del Host solo a través de su límite de confianza o autenticado por separado hasta que se añadan autorización y atribución.
- **Límites de catálogo y de fila** — una petición en frío escanea el catálogo completo de instantáneas de la Sesión porque la persistencia no tiene una operación de metadatos de búsqueda por id. `maxNoteBytes` limita una nota, pero el recuento de elementos y los bytes retenidos agregados de una fila de Sesión no tienen tope; una lectura de metadatos indexada y un límite de fila propiedad del despliegue quedan diferidos hasta que un Consumer concreto defina su política.
