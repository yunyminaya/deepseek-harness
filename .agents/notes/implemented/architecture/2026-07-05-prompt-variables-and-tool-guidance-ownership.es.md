# Agent Note: Variables de prompt y propiedad de la guía de herramientas

Status: implemented

[English](2026-07-05-prompt-variables-and-tool-guidance-ownership.md) | Español

## Problema

El prompt del sistema ensamblado tenía cuatro defectos, todos de la misma familia: hechos que el harness (marco de trabajo para agentes) ya conoce se reafirmaban a mano en algún otro sitio y se desincronizaban.

**El modelo no podía conocer su propio nombre.** `AgentOptions.model` impulsa cada petición, pero ningún texto del prompt lo llevaba — y nada PODÍA llevarlo: las secciones de `dsh-system-prompt` eran globales al contexto mientras que el nombre del modelo es por-agent, y `assemble()` no aceptaba ninguna entrada por-agent.

**La guía de herramientas era prosa escrita a mano en leaf YAML.** La guía de uso de shell/subagent/todo_write vivía en las cadenas de persona del coding agent y de ACP (Agent Client Protocol) — dos copias desincronizadas (la de ACP ya estaba abreviada) — mientras que `dsh-tool-fs` y `dsh-tool-web` aportaban su guía como contribuciones de `ctx.systemPrompt.section()`. Cargar o quitar un plugin de herramientas significaba editar a mano la persona de cada despliegue, y el antiguo banner de bienvenida del terminal también enumeraba a mano el conjunto de herramientas.

**La persona se renderizaba después de la guía de herramientas.** El agent loop (bucle del agente) unía `agent.options.systemPrompt` DESPUÉS de las secciones ensambladas, de modo que el modelo leía «Use the read tool…» antes de «You are a coding agent» — al revés de la convención identity-first (Claude Code, Codex) — y era una segunda vía de composición aparte de la tubería de secciones.

**La descripción de la herramienta fork era falsa.** `dsh-tool-subagent` tenía fija una descripción escrita para la semántica de spawn — «a separate agent that works in its own context … it does not see this conversation» — y la instancia `subagent_fork` (cuyo hijo hereda los turnos completados del padre) recibía las mismas palabras; la prosa YAML corregía la mentira fuera de banda. Pariente menor: `PromptSection.name` estaba documentado como «(diagnostics / dedup)», pero los duplicados se aceptaban en silencio.

## Decisión

**Un principio: cada hecho del prompt tiene exactamente un dueño.** El nombre del modelo y el workspace son hechos de config/sesión → el harness los expone como variables y la persona las referencia. La semántica de cada herramienta y cuándo usarla → la `description` de la herramienta. Los hábitos entre llamadas que una descripción no puede transportar → la sección de prompt del paquete de la herramienta. El nombre del producto y la línea de identidad del SDK → la sección estática `harness:identity`. El rol y el comportamiento del despliegue → la persona del despliegue.

### Contexto de assemble

`SystemPrompt.assemble(context)` acepta un `AssembleContext` extensible por fusión de declaraciones. `dsh-system-prompt` declara el selector opcional `scope` usado para el enrutamiento por alcance, mientras que `dsh-agent` fusiona por declaración el campo tipado opcional `agent` sobre él (una arista a nivel de tipos `agent → system-prompt`, sin ciclo de dependencias en tiempo de ejecución). El loop llama a `assembleContextFor(agent)` en cada paso para que ambos campos identifiquen al mismo agent (agente); los proveedores de texto de sección pueden leer ese contexto, y el waterfall (cascada de eventos) de `system-prompt/assemble` lo recibe para que un listener pueda filtrar o ampliar por agent.

### Variables de prompt

Los plugins registran valores `{{name}}` mediante `ctx.systemPrompt.variable(name, provider)`. El ensamblado los resuelve en el mapa de variables visible en el waterfall. El renderizado rechaza las referencias desconocidas de propiedad propia, los providers registrados que devuelven `undefined`, las referencias completas malformadas y las referencias desbalanceadas que aún contienen un `}}` de cierre; un `{{` suelto sin pareja permanece como prosa, y los valores sustituidos no se vuelven a escanear. El registro rechaza los nombres de variable inválidos o duplicados, y los nombres de sección son únicos.

`dsh-agent-loop` registra las dos variables integradas, ambas proyecciones puras del agent del contexto: `model` (= `options.model`) y `cwd` (= `session.header.cwd`). Las personas de ejemplo escriben `powered by the {{model}} model` — el nombre del modelo se afirma una sola vez, en la clave de config `model:`. `{{cwd}}` solo se demuestra en el ejemplo de ACP: cada sesión ACP lleva el cwd del cliente, mientras que los agents stdio precreados por config no tienen ninguno (una persona que reclame `{{cwd}}` allí falla el turno — a propósito). Las variables permanecen en el plugin del loop (a diferencia de las secciones siguientes): son hechos de tiempo de ejecución de los agents que ESTE loop impulsa, y un loop de reemplazo aporta las suyas.

### La persona como sección de orden 0

`dsh-system-prompt` es dueño de `harness:identity` en el orden `-100` y de la `deployment:persona` configurada en el orden 0, de modo que ambas sobreviven a un loop de reemplazo. El renderizado del prompt tiene una sola vía, `renderPrompt(assembly)`, y la cabecera de la petición enrutada registra por tanto el prompt exacto que luego reproduce `ctx.tokenMeter` para la presión de compactación. Una `deployment:persona` con alcance de agent sombrea el valor global por defecto y permite que los providers de subagente instalen una persona antes de la publicación. Las bandas de orden convencionales son identidad `-100`, persona `0` y guía de herramientas `100–199`.

### Propiedad de la guía de herramientas

La semántica de cada herramienta y la guía de selección viven en las descripciones de las herramientas. Las secciones de prompt solo transportan hábitos entre llamadas, como comprobar los marcadores de salida de bash o preferir las herramientas de sistema de archivos a los comandos de shell. `todo_write` y las herramientas de subagente no necesitan sección porque sus descripciones contienen el contrato completo. Las personas del despliegue contienen solo rol y comportamiento.

### El descriptor de historial de conversación del subagente

`SubagentProvider.inheritsParentContext` describe la siembra de la conversación, no el alcance, los servicios, las herramientas ni la autoridad. spawn y ACP lo ponen a `false`; fork lo pone a `true`. `dsh-tool-subagent` deriva de esa bandera sus descripciones de herramienta y de parámetro de prompt, incluido que fork hereda los turnos completados pero no el turno en curso. Los eventos de ciclo de vida del provider mantienen ese redactado sincronizado con el registro reactivo de providers; su fundamento vive en la [Agent Note de los eventos de ciclo de vida del provider](2026-07-05-subagent-provider-lifecycle-events.es.md).

## Alternativas consideradas

- **El loop compone una línea de identidad él mismo** — fija prosa orientada al modelo en el único paquete que debe mantenerse delgado («plugins, not loop changes»), y fuera de la tubería de secciones sería una segunda vía de composición. (La identidad SÍ viaja como literal de código — pero como sección ordinaria registrada por `dsh-system-prompt`, cuyo waterfall de `system-prompt/assemble` sigue siendo la válvula de escape para un despliegue que deba eliminarla.)
- **Inyectar el nombre del modelo vía el waterfall de `agent/request`** — el texto del prompt se compondría en dos sitios, y la persona renderizada antes podría no coincidir con la cabecera final enrutada. El plugin de petición dueño del enrutamiento tardío también debe ser dueño de cualquier afirmación previa del prompt sobre ese modelo.
- **Escribir a mano el nombre del modelo en cada persona** — duplica la clave `model:` una línea más arriba y miente en silencio tras una edición de config; exactamente la enfermedad que esta decisión cura.
- **Interpolación permisiva (dejar las referencias desconocidas verbatim o sustituir vacío)** — un error tipográfico envía `{{modle}}` (o un hueco) al modelo y nadie lo nota hasta la revisión del transcript.
- **Redactado de subagente por instancia en config** — devuelve la prosa orientada al modelo a cada despliegue × instancia, reviviendo la deriva de la guía escrita a mano en leaf YAML. **Clavar el redactado al NOMBRE del provider** — `providerName` es en sí config, así que un provider renombrado recibe en silencio las palabras equivocadas.
- **Resolver el provider en el momento de `apply` (un requisito de orden de carga)** y **redactado de subagente solo en sección (resuelto perezosamente en assemble)** — las alternativas a los eventos de ciclo de vida del provider; ambas se rechazan en [la Agent Note de los eventos de ciclo de vida del provider](2026-07-05-subagent-provider-lifecycle-events.es.md).

## Fuera de alcance

- Más variables (`date`, plataforma, estado de git) — el registro hace de cada una una contribución de una línea por parte del plugin dueño del hecho; ninguna se reclama aquí.
- Un `cwd` de config para los agents stdio precreados (permitiría que la persona stdio usara `{{cwd}}` y particionara la persistencia por ruta real) — diferido hasta que se reconsidere la historia del cwd de sesión.

## Invariantes publicados

- El prompt del tui-agent renderiza la identidad, la persona con el modelo interpolado y luego la guía de fs/shell/web a través de una sola vía de ensamblado.
- Las descripciones de fork y de subagente nuevo reflejan si el provider hereda los turnos de conversación completados; la herramienta aparece, desaparece y se reformula con los cambios de ciclo de vida del provider.
- Las referencias a variables desconocidas, sin valor, malformadas o desbalanceadas nombran la sección y lanzan una excepción; los registros duplicados de sección, variable y herramienta también lanzan.
- La reproducción de instantáneas es independiente del prompt: clava los flujos de chunks registrados por turno y paso sin comparar la petición saliente.

## Consecuencias

- Cada hecho del prompt ensamblado tiene ahora exactamente un dueño, y la prosa de herramientas mantenida a mano en leaf YAML ha desaparecido: cargar o quitar un plugin de herramientas ya no implica editar la persona de ningún despliegue.
- `{{model}}` refleja `AgentOptions.model` en el momento del ensamblado. Un plugin que cambia de modelo en el waterfall de `agent/request` deja obsoleta la afirmación del prompt para ese paso, y uno que SUMINISTRA el modelo allí (`options.model` sin definir — el fallback documentado del loop) deja la variable sin valor en el render, haciendo fallar a una persona con `{{model}}` antes de que el waterfall se ejecute. Ambos tienen el mismo remedio, y es la propia regla de propiedad: el plugin dueño del hecho del modelo de enlace tardío lo afirma pronto en el waterfall de `system-prompt/assemble` (`assembly.variables['model'] = …`) — un dueño, ambas afirmaciones; una prueba del loop fija la vía de suministro de extremo a extremo. Aceptado.
- Mientras un provider vinculado está ausente (aún no activado, descargado, a mitad de una recarga de HMR (hot module replacement)), la herramienta de subagente no existe y una petición del modelo en esa ventana simplemente carece de ella. Ese es el estado honesto — la alternativa era una herramienta registrada cuya descripción o ejecución no fuera de fiar.
- La estrictez significa que una persona puede hacer fallar un turno en el render (p. ej. `{{cwd}}` en una sesión sin cwd). El fallo está contenido — el turno termina en `error`, el loop sobrevive — y es un error de autoría que QUEREMOS que sea ruidoso.
- Aún no hay sintaxis de escape para un `{{name}}` literal en la prosa del prompt; añade una si algún prompt real llega a necesitarla.
