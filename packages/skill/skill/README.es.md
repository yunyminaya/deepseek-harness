# @deepseek-ai/dsh-skill

[English](README.md) | Español

Registro puro de providers de skill (destreza) para agent (agente).

Este paquete es dueño de la interfaz `ctx.skills`. No sabe si los skills provienen de archivos locales, datos de plugin integrados, HTTP u otro backend; los providers registran esas fuentes con `ctx.skills.registerProvider(...)`. La implementación local publicada es [`@deepseek-ai/dsh-skill-filesystem`](../skill-filesystem).

El registro se superpone por host y por scope sobre [`@deepseek-ai/dsh-scope`](../../core/scope), la forma que estableció el registro de herramientas: un registro archiva en la capa del scope del contexto llamante — las filas de host y los plugins de repositorio caen en la capa global, un plugin montado por la composición permanente de un preset de agent cae en la capa de ese preset — y una lectura fusiona la capa global con la cadena del scope de visión, ganando la capa más cercana un nombre duplicado directamente mientras que el rango solo decide los duplicados dentro de una misma capa.

## Servicio: `SkillRegistry` (clave de ctx: `skills`)

### API pública

- `ctx.skills.registerProvider(create): () => void` Llama a una fábrica de provider síncrona con `{ signal, invalidate }` y registra después su resultado de solo lectura por `provider.name`, único dentro de la capa del contexto llamante. Los nombres duplicados en una capa lanzan una excepción, `runtime` está reservado y un registro fallido aborta la señal. El disposer exacto de Cordis anula el registro del provider, aborta la señal y conserva el teardown compuesto ordenado.
- `ctx.skills.snapshot({ cwd?, signal?, scope? })` Devuelve la observación `{ skills, complete }` neutral a la invocación para las capas fusionadas del scope de visión. `complete` es false cuando cualquier provider rechaza o informa explícitamente de un descubrimiento incompleto, o cuando una segunda revisión del catálogo compite con el reintento acotado; los candidatos aportados por esa observación permanecen en este resultado, que nunca se cachea.
- `ctx.skills.list({ cwd?, signal?, scope? })` Toma prestadas las opciones de vista de solo lectura y devuelve todos los resúmenes ganadores del espacio de trabajo actual, fusionados entre la capa global y la cadena del scope de visión y ordenados por nombre. Los consumidores aplican `isModelInvocable(skill)` o `isUserInvocable(skill)` en su propio límite.
- `ctx.skills.get(name, { cwd?, signal?, scope? })` Usa las mismas opciones de solo lectura y el candidato ganador para el descubrimiento y la carga, vuelve a comprobar la cancelación después del descubrimiento o de un acierto de caché, hace competir la carga del provider contra la señal, valida la definición cargada y la devuelve independientemente de la política de invocación.
- `ctx.skills.register(skill): () => void` Registra un skill integrado de runtime de solo lectura en la capa del contexto llamante, añadiendo la política de todo invocable y `provider: "runtime"` cuando se omite. Los registros de runtime con el mismo nombre en una capa ganan por primer llegado: un duplicado registra un aviso y recibe un disposer no operativo. Los registros con éxito devuelven el disposer exacto de Cordis para el teardown compuesto ordenado.

### Eventos

- `skills/change` es una notificación de invalidación sin filtrar emitida después de que se registre o se deseche una contribución de provider o runtime, y después de que el control de registro de un provider activo invalide. No lleva catálogo ni diff: cada consumidor vuelve a obtener `snapshot()` con sus propias opciones de búsqueda. Las excepciones de los listeners y las promesas rechazadas se registran y no pueden vetar la mutación del registro ni privar de eventos a listeners posteriores.

### Config

| Campo | Predeterminado | Significado |
|---|---|---|
| `collectCacheMaxEntries` | `128` | Número máximo de catálogos completados por cwd/provider mantenidos en memoria. |

### Política de invocación

`SkillSummary.invocation` es un objeto de política tipado obligatorio cuyos booleanos positivos `modelInvocable` y `userInvocable` describen las dos superficies de forma independiente. Los providers devuelven esta forma resuelta en cada candidato y definición; solo la entrada `SkillRegistration` puede omitirla, en cuyo caso `register()` suministra `{ modelInvocable: true, userInvocable: true }`. El registro conserva las cuatro combinaciones para que un solo resultado de descubrimiento pueda servir a las herramientas orientadas al modelo, a los comandos orientados a humanos y a los llamantes internos de confianza sin mezclar sus catálogos.

| Política | Modelo | Usuario |
|---|---|---|
| `{ modelInvocable: true, userInvocable: true }` | incluido | incluido |
| `{ modelInvocable: true, userInvocable: false }` | incluido | excluido |
| `{ modelInvocable: false, userInvocable: true }` | excluido | incluido |
| `{ modelInvocable: false, userInvocable: false }` | excluido | excluido |

### Renderizado compartido orientado al modelo

`renderSkillContent(skill)` renderiza un skill cargado como el bloque canónico `<skill_content>` (atributo `name` escapado, pistas de recursos, cuerpo verbatim). Es la única verdad para ambas rutas de carga: `dsh-tool-skill` lo devuelve como resultado de la herramienta `skill` y lo inyecta en el límite del gesto explícito del usuario, de modo que el modelo ve una sola forma independientemente de quién inicie la carga. `escapeText` se exporta junto a él para los consumidores que incrustan prosa en el mismo marco de marcado. El paquete también declara el tipo `MessageSource` `skill-invocation` ({ name, form: 'instructions' }) que la inyección explícita del usuario estampa en sus mensajes — los consumidores de transcript (transcripción) presentan la invocación a partir de estos metadatos en lugar de reanalizar el cuerpo.

`isModelInvocable(skill)` y `isUserInvocable(skill)` leen directamente el campo positivo correspondiente. `ctx.skills.get()` sigue siendo la primitiva de carga de confianza y neutral a la política, de modo que todo consumidor orientado al usuario o al modelo debe aplicar el predicado que corresponde a su superficie antes de exponer o cargar un skill.

## Contrato del provider

Una fábrica de provider se ejecuta de forma síncrona y recibe un único control con ámbito de registro. `control.signal` aborta cuando el registro falla o se desecha; `control.invalidate()` limpia solo los catálogos completados mientras ese registro exacto siga activo, de modo que las devoluciones de llamada tardías no puedan afectar a un reemplazo con el mismo nombre. Los providers inmutables pueden ignorar el control. La configuración remota, la autenticación y el descubrimiento pertenecen a la llamada `list(options)` en espera del provider. Devolver un array es una abreviatura de descubrimiento completo; un provider que recopiló candidatos utilizables pero no pudo establecer una observación autoritativa devuelve `{ candidates, complete: false }`. Los objetos de provider, las opciones de búsqueda, los candidatos y las definiciones se toman prestados de solo lectura, sin clonar ni reenlazar. Los providers deben respetar `options.signal`; el registro también deja de esperar descubrimientos o cargas poco cooperativos después de la cancelación.

El registro valida los candidatos antes de cachearlos y las definiciones antes de devolverlas. El provider ganador recibe el mismo candidato y el `locator` opaco que devolvió de `list()`, lo que permite manejadores de archivo, URL, id o versión específicos del backend. Los llamantes y los providers deben conservar el contrato de solo lectura.

Las violaciones del contrato fallan rápido. Un `list()` de provider rechazado se trata como un fallo de fuente transitorio y se omite. Una observación incompleta explícita sigue aportando sus candidatos para `list()` y `get()`, pero hace que la instantánea agregada sea incompleta e incacheable. Un cambio de revisión de provider o runtime descarta un resultado en vuelo y reintenta una vez. Si el reintento también queda superado, sus candidatos se devuelven incompletos y sin cachear, de modo que un provider que invalida continuamente no pueda monopolizar al llamante. Dentro de una capa, los nombres duplicados se resuelven por rango, luego por orden de registro del provider y después por orden local del provider; entre capas, la entrada del scope más cercano gana el nombre. Los resúmenes se ordenan por nombre de skill.

Las definiciones se cargan de forma progresiva. `get()` pide el cuerpo al provider ganador en cada llamada en lugar de cachearlo en este registro. Si la definición devuelta tiene un nombre distinto del candidato seleccionado, la selección obsoleta se rechaza y el registro invalida internamente a ese provider exacto para que la siguiente instantánea redescubra su catálogo.

## Skills en runtime

`ctx.skills.register(...)` es una conveniencia para skills de runtime integrados. Los skills de runtime usan el rango `250`: los providers de proyecto pueden anularlos, mientras que ellos anulan las raíces personalizadas y de usuario del provider local publicado. Las definiciones de runtime y los metadatos de recursos anidados se toman prestados de solo lectura; el servicio materializa una definición de nivel superior para suministrar los predeterminados omitidos de invocación y provider. El registro es de primer llegado dentro de las contribuciones de runtime, de modo que una contribución duplicada no puede retirar a la activa a través de su disposer.

## Límite del Consumer

El registro no renderiza guía para el modelo ni registra herramientas orientadas al modelo. [`@deepseek-ai/dsh-tool-skill`](../tool-skill) consume `ctx.skills` para proporcionar los catálogos durables de sesión y la herramienta `skill`, de modo que los providers permanecen independientes del comportamiento orientado al modelo.

## Experiencia del modelo

De forma indirecta, a través de `dsh-tool-skill`, que renderiza los resúmenes de los providers en mensajes durables de catálogo inicial o de reemplazo y las instrucciones cargadas en resultados de herramienta retenidos.

#### Efecto en la caché KV

Sin efecto directo en el prompt. El Consumer nombrado es dueño del catálogo inicial durable y de los reemplazos de solo añadido después de la invalidación.

## Limitaciones conocidas y trabajo diferido

- **La invalidación la impulsa el provider** — el registro no tiene TTL y no puede inferir que una fuente remota arbitraria cambió; cada provider mutable debe retener y llamar a su capacidad `invalidate()` con ámbito de registro desde su propio mecanismo de observación.
- **Los providers se consultan en secuencia** — un provider cooperativo lento retrasa a todos los registrados después de él; la cancelación detiene la espera del llamante, pero no puede terminar el trabajo que un provider poco cooperativo sigue ejecutando.
- **Las observaciones incompletas no se retienen** — los providers rechazados se omiten y los candidatos suministrados explícitamente solo están disponibles para la búsqueda actual; el registro no es dueño ni de un último catálogo bueno ni de diagnósticos por provider.
- **La resolución de duplicados es de primer llegado** — los candidatos posteriores de menor prioridad dentro de una capa se registran y ocultan, y una capa más cercana ensombrece a una más lejana en silencio; no hay API para inspeccionar todas las definiciones ensombrecidas.
