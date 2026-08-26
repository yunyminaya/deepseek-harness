# @deepseek-ai/dsh-tool-skill

[English](README.md) | Español

El catálogo de skills (destreza) orientado al modelo y la herramienta `skill`.

Requiere `ctx.agents`, `ctx.tools` y `ctx.skills` (`inject: ['agents', 'tools', 'skills']`).

## Ciclo de vida del catálogo

En cada `agent/pre-step` elegible, el plugin llama a `ctx.skills.snapshot()` para el cwd de la sesión llamante, reenvía la señal de aborto del pre-step al descubrimiento, aplica la visibilidad exacta de la herramienta `skill` y renderiza las entradas ordenadas `name` y `description`. Cuando no existe un catálogo previo y esa vista no está vacía, añade un `<system-reminder>` durable inicial de rol de usuario a una decisión `enter` posterior. Los mensajes del catálogo contienen solo esos resúmenes; los cuerpos de skill, las rutas, las fuentes, los providers y las pistas `whenToUse` quedan fuera del catálogo.

Cada mensaje de catálogo lleva la fuente `skill-catalog`: un contexto de forma `catalog` cuyas `entries` registran exactamente los pares `name` y `description` que publicó, más `update` en un reemplazo. El resumen cubre esas entradas durables, no la prosa renderizada, de modo que el marco circundante `<system-reminder>` no puede decidir si hace falta republicar y los consumidores nunca reanalizan el bloque `<available_skills>`. El plugin escanea los eventos durables de sesión hacia atrás sin copiarlos y deriva la línea de base de comparación del mensaje `skill-catalog` visible más reciente que puede leer; los registros ilegibles y ajenos se omiten. Cuando el resumen cambia, la decisión `enter` posterior recibe un mensaje durable de rol de usuario con el catálogo de reemplazo completo; un reemplazo vacío retira explícitamente los nombres anteriores. Si no queda ningún catálogo visible pero existe un catálogo histórico reconocible, la compactación lo ocultó y la siguiente observación completa restablece el catálogo actual. Una instantánea de provider incompleta no emite nada y conserva la última vista de modelo buena para reintentar en el siguiente pre-step. Si no existe un catálogo previo y la vista actual está vacía, no hace falta ningún tombstone.

El catálogo se omite cuando no hay skills invocables por el modelo disponibles inicialmente, y también cuando la vista de herramientas de ese agent elimina la herramienta `skill` publicada o resuelve en su lugar una sombra con ámbito de mismo nombre. La identidad se compara contra la definición que registró este plugin en lugar de una búsqueda de su propio nombre, de modo que el plugin funciona montado globalmente o dentro de la composición de un solo agent, donde `register()` archiva solo en la capa de ese agent. Los cambios de visibilidad participan en el resumen, manteniendo alineados la guía de prompt, el schema visible para el modelo y el despacho ejecutable.

`catalogDescriptionMaxLength` controla las descripciones de catálogo normalizadas; el renderizado las escapa en XML. Su predeterminado es `500` y los valores deben ser enteros de al menos `3`, lo que reserva espacio para una elipsis de truncamiento. La [Agent Note de hot-refresh del catálogo de skills](../../../.agents/notes/implemented/feature/2026-07-27-skill-catalog-hot-refresh.es.md) es dueña del catálogo inicial durable y del ciclo de vida de reemplazo.

## Herramienta: `skill`

| Argumento | Tipo | Notas |
|---|---|---|
| `name` | string (obligatorio) | Nombre de skill exacto en kebab-case de la lista de skills disponibles. |

La ejecución usa el `session.header.cwd` del agent llamante para que los providers sensibles al espacio de trabajo resuelvan el skill ganador. Una llamada con éxito devuelve el canónico `{ name, provider, resourceBase?, content }`, excluyendo el rango del catálogo y la maquinaria interna del provider; su renderizador Native produce un único resultado de texto que contiene `<skill_content name="...">`, `<skill_resources>` y `<skill_instructions>`.

La guía de recursos resuelve solo las rutas o URLs referenciadas explícitamente por las instrucciones contra `resourceBase`; los scripts, references y assets se cargan bajo demanda, y el resultado no enumera un directorio de skill. Los providers locales pueden suministrar un directorio, mientras que los remotos o integrados pueden suministrar una URL o una guía de carga opaca.

Un nombre no resuelto informa de que el skill es desconocido o ya no está disponible. Los nombres no válidos y los skills cuyo `invocation.modelInvocable` es `false` producen resultados de error distintos. `invocation.userInvocable` no restringe esta herramienta orientada al modelo.

La ejecución de la herramienta no añade un mensaje de contexto sintético. Su resultado recién cargado ya se registra como resultado de la herramienta y queda disponible para el siguiente paso del modelo sin duplicar el cuerpo. Solo la proyección del catálogo añade resúmenes de reemplazo.

## Experiencia del modelo

### Catálogo de sesión

#### Lo que ve el modelo

Si existen skills invocables por el modelo y esta herramienta `skill` exacta es visible, el agent recibe la plantilla de catálogo siguiente como mensaje durable de rol de usuario antes de la primera solicitud, con una entrada dependiente de los datos por skill ordenado. Los cambios posteriores de pertenencia, descripción o visibilidad añaden un reemplazo completo con el mismo envoltorio `<available_skills>`; eliminar todos los skills añade un envoltorio vacío con una instrucción explícita de no usar nombres anteriores. La frase final de la plantilla es la regla contra la doble carga: el límite del gesto explícito del usuario (el listener de pre-step siguiente) inyecta la misma salida de `renderSkillContent` (compartida desde `@deepseek-ai/dsh-skill`) en línea, y el catálogo dice al modelo que siga ese bloque en lugar de volver a cargar el skill a través de la herramienta; la plantilla de catálogo de reemplazo lleva la misma frase en ambos brazos, incluido el catálogo vaciado.

##### Plantilla de catálogo de skills

```markdown
<system-reminder>
A skill is a reusable set of task-specific instructions. The following skills are available in this session:

<available_skills>
- `<name>`: <normalized-and-capped-description>
</available_skills>

If the user names a skill, or the task clearly matches a skill's description, call the `skill` tool with the exact skill name before taking task actions. Load all applicable skills, then follow their full instructions. This catalog contains summaries only; do not infer or follow a skill's instructions until it has been loaded.
A user may also invoke a skill directly; its <skill_content> block then appears in this conversation. Follow it, and do not call the `skill` tool again for that skill.
</system-reminder>
```

#### Efecto en tokens

El coste de entrada repetido escala con el número de skills y con `catalogDescriptionMaxLength`; no se envían tokens de catálogo inicial cuando la lista está vacía o la herramienta está oculta o ensombrecida. Cada cambio real de catálogo añade un mensaje retenido completo de reemplazo.

#### Efecto en la caché KV

El catálogo durable inicial se añade después del prefijo reutilizable existente. Los cambios dinámicos son historial de solo añadido después de ese catálogo, de modo que los tokens reutilizables anteriores permanecen intactos mientras cada catálogo recién añadido y los turnos posteriores forman un sufijo nuevo. Una instancia nueva o reanudada con un resumen cambiado puede afectar a la reutilización de la caché desde la posición del catálogo recién añadido.

### Schema de la herramienta

#### Lo que ve el modelo

El modelo ve el [schema `skill`](../../../docs/tool-catalog.es.md#deepseek-aidsh-tool-skill) generado.

#### Efecto en tokens

Coste fijo de schema por solicitud donde la herramienta es visible.

#### Efecto en la caché KV

Estable en el prefijo mientras la definición y la visibilidad de la herramienta no cambien. El ensombrecimiento, las restricciones o los cambios del ciclo de vida del plugin pueden invalidar la reutilización desde este schema.

### Resultado de la herramienta

#### Lo que ve el modelo

Una llamada con éxito usa la plantilla de resultado y la guía de recursos gestionada por el provider —de directorio, URL u opaca— siguiente.

##### Plantilla de resultado de skill

```markdown
<skill_content name="<escaped-name>">
<skill_resources>
<resource-guidance>
</skill_resources>

<skill_instructions>
<provider-owned-instruction-body>
</skill_instructions>
</skill_content>
```

##### Guía de recursos gestionada por el provider

```markdown
Resources for this skill are managed by provider "<provider>".
Load referenced resources only as needed.
```

##### Guía de recursos de directorio

```markdown
Base directory for this skill: <path>
Resolve relative paths mentioned by this skill against the base directory before using them. Load referenced resources only as needed.
```

##### Guía de recursos de URL

```markdown
Base URL for this skill: <url>
Resolve relative URLs mentioned by this skill against the base URL before using them. Load referenced resources only as needed.
```

##### Guía de recursos opaca

```markdown
Resources for this skill: <description>
Load referenced resources only as needed.
```

#### Efecto en tokens

Las instrucciones cargadas son tokens de resultado de herramienta dependientes de los datos, que se reenvían en pasos posteriores hasta la compactación; no se hace ninguna copia duplicada de `agent.inject()`.

#### Efecto en la caché KV

Solo añadido; el contenido recién visible sigue el prefijo de solicitud reutilizable y no invalida las entradas existentes de la caché KV.

### Errores de la herramienta

#### Lo que ve el modelo

Las selecciones no válidas u obsoletas devuelven exactamente `Error: invalid skill name "<name>"`, `Error: skill "<name>" is unknown or no longer available` o `Error: skill "<name>" is not available for model invocation`. El texto de búsqueda lanzado por el provider depende de los datos y recibe el mismo envoltorio `Error: <message>`.

#### Efecto en tokens

Solo una llamada fallida añade estos tokens retenidos.

#### Efecto en la caché KV

Solo añadido; el contenido recién visible sigue el prefijo de solicitud reutilizable y no invalida las entradas existentes de la caché KV.

### Inyección de invocación explícita del usuario

#### Lo que ve el modelo

Un token `/name` delimitado por espacios en cualquier lugar de un mensaje de usuario reclamado, que nombre un skill invocable por el usuario en el catálogo del espacio de trabajo, inyecta la representación completa `<skill_content>` de ese skill (la forma exacta de la plantilla de resultado anterior) como contexto de instrucciones de rol `user` añadido después de todas las demás inyecciones de ese paso — en segundo plano primero, el material sobre el que actuar al final. Solo se escanea la entrada directa del usuario, la comprobación se ejecuta sobre la definición cargada, y los nombres desconocidos o deshabilitados para el usuario permanecen como prosa ordinaria. Este es el único punto de entrada de los skills con `disable-model-invocation`, que el catálogo y la herramienta `skill` nunca exponen; la frase final del catálogo dice al modelo que siga el bloque inyectado en lugar de volver a cargarlo.

#### Efecto en tokens

Cada gesto añade un cuerpo de skill renderizado a ese turno como contexto inyectado — el mismo tamaño que el resultado de la herramienta para el mismo skill, pagado de forma determinista a petición del usuario en lugar de a discreción del modelo. Los gestos repetidos de un skill dentro de un mismo paso se inyectan una sola vez.

#### Efecto en la caché KV

Solo añadido; la inyección aterriza después del prefijo de solicitud reutilizable dentro del lote de mensajes del paso y no invalida las entradas existentes de la caché KV.

## Limitaciones conocidas y trabajo diferido

- **El catálogo omite `whenToUse`, la fuente y los metadatos del provider** — el enrutamiento se basa solo en el nombre y en una descripción con tope; `whenToUse` sigue siendo metadatos del provider y el envoltorio cargado tampoco lo renderiza.
- **Los cuerpos de instrucciones cargados no tienen tope de tamaño** — un provider puede devolver un skill lo bastante grande como para consumir contexto sustancial del siguiente paso; solo se truncan las descripciones del catálogo.
- **Los recursos son guía, no adjuntos** — la herramienta informa de un directorio/URL/pista opaca base, pero ni enumera ni obtiene los archivos referenciados para el modelo.
- **La carga es texto de una sola vez** — no hay manejador parcial, en streaming ni de contenido cacheado cuando un provider remoto es lento o el cuerpo de un skill es grande.
- **El reemplazo del catálogo es de lista completa** — un nombre o descripción cambiados añaden todos los resúmenes actualmente visibles; esto mantiene explícita la retirada de nombres obsoletos, pero cuesta tokens proporcionales al catálogo.
- **Los cuerpos no tienen versiones** — las ediciones solo de cuerpo no cambian el resumen del catálogo ni notifican al modelo; una llamada posterior de herramienta lee el contenido actual del provider mientras los resultados de herramienta anteriores siguen siendo hechos históricos.
