# @deepseek-ai/dsh-client-ui-model-selection

[English](README.md) | [中文](README.zh.md) | Español

Plugin de selección de modelo, mitad navegador: DOS entradas sobre UN directorio por sesión propiedad de `ModelDirectoryResolver` (`ctx.modelDirectories`). Para sesiones ordinarias, la contribución popupSelect `/model` (registrada a través de `ctx.commandUi`) y el asiento con nombre `conversation.input.model` del compositor cargan ambos el directorio consultivo de la sesión a través de `session.models` y envían a través de `session.selectModel` mediante la misma instancia de `ModelDirectory`. El disparador compacto del compositor abre un menú de dos niveles Model/Effort: los modelos permanecen agrupados por provider, mientras que el modelo exacto seleccionado aporta los nombres de effort de su adaptador, sus descripciones y su valor por defecto. `/model` aplica el effort por defecto del modelo seleccionado, y el compositor puede elegir entonces cualquier effort anunciado.

El `ModelSelection` de provider/modelo/razonamiento reportado por el Host es el único hecho de selección, pero solo se hace eco cuando el par exacto de provider/modelo permanece en los grupos anunciados; una fila de catálogo ausente deja la selección enrutable intacta mientras el disparador solicita `Select model`, no se sintetiza ninguna fila obsoleta y no se muestra ninguna fila de Effort hasta que el usuario elige un modelo anunciado. Las cargas de directorio y las selecciones comparten un contador de generación para que una respuesta antigua nunca sobrescriba una más nueva; un reinicio de la conexión descarta cada proyección residente y vuelve a obtener la selección restaurada por el Host antes de mostrarla. Los fallos de metadatos locales al provider se listan en línea mientras los grupos utilizables siguen seleccionables, y los fallos de selección conservan la selección y el directorio anteriores.

Cuando el Host reporta que ningún adaptador sirve la ruta de la sesión (`session.models.routable`), este plugin eleva un bloque de compositor a través de `ctx.conversation.blocks` y la entrada queda inerte con el texto propio de este plugin; la recuperación lo limpia sin recargar. Sigue a `routable` y nada más: un `null` — antes de la primera carga, o después de una fallida — nunca bloquea, o un Host lento bloquearía un compositor que funciona, y la pertenencia al catálogo tampoco bloquea nunca, porque una ruta que sirve un modelo que dejó de anunciar falta de los grupos pero es perfectamente utilizable. El fallback `Select model` del propio disparador cubre aún ese caso, que es de visualización, no un gate.

Los directorios son por sesión, se resuelven de forma diferida a través de `ctx.modelDirectories.directoryFor(sessionId)` y se disponen con el ámbito de la sesión. Las sesiones de subagente direccionadas no exponen ninguna de las dos entradas, y su directorio rechaza cargas, selecciones y refrescos por reconexión, porque los RPC de modelo ordinarios ligados al Agent activarían el historial persistido del hijo fuera de la ruta de continuación del padre directo.

Cada directorio residente vuelve a obtener directamente ante los eventos de dueño reenviados `llm/adapters-updated` y `settings/document-updated`. La topología de providers, los catálogos de providers y la selección por defecto convergen así sin que el Host ni el runtime del cliente deriven un alias separado de cambio de modelo.

Los exports de `/client` son el cuerpo del plugin (`apply`/`inject`), `ModelDirectoryResolver`, `ModelDirectory` con sus campos de estado y el tipo de cara inyectada del asiento.

## Experiencia de modelo

Indirectamente, a través del RPC `session.selectModel` disponible para las sesiones ordinarias: ambas entradas envían el `ModelSelection` completo que el Host captura en el siguiente límite de ensamblaje del prompt, de modo que la siguiente solicitud usa el provider, el modelo y el effort seleccionados mientras un paso en ejecución conserva su selección ensamblada; la selección se vuelve duradera solo cuando la cabecera de la solicitud existente registra una solicitud que la consume, y la interacción con el menú no añade contenido al prompt.

#### Efecto en KV Cache

Cambiar la ruta puede reducir o invalidar la reutilización de caché del lado del provider para solicitudes posteriores; el prefijo del prompt en sí permanece intacto.

## Limitaciones conocidas y trabajo diferido

- **Sin selección en el momento de creación ni para subagentes direccionados** — ambas entradas requieren el Agent de una sesión ordinaria existente; no hay elección de modelo en fase de borrador que plegar en la creación de la sesión, y la continuación de subagentes no expone deliberadamente ningún contrato independiente de selección de modelo.
- **Los nombres de directorio son solo de presentación** — la selección y la persistencia usan ids de provider/modelo/effort; un provider cuya búsqueda de catálogo o de metadatos de modelo exacto falla se lista como fila de fallo no seleccionable hasta recargar.
- **Sin entrada de effort arbitraria** — el compositor ofrece solo los niveles anunciados por el adaptador del modelo exacto; un adaptador sin metadatos de razonamiento deja la fila de Effort ausente.
