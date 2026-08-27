# Agent Note: Sistema de skills — instrucciones con revelación progresiva para agents

Status: implemented

[English](2026-07-05-skill-system.md) | Español

## Problema

Los productos de agents han convergido en un patrón de skills: mantener pequeño el prompt de la petición listando solo los paquetes de instrucciones disponibles, y cargar el cuerpo completo cuando el modelo decide que una tarea coincide. Codex, Claude Code, OpenCode y Kimi Code difieren en detalles, pero todos separan los metadatos de descubrimiento de las instrucciones completas, de modo que un workspace puede portar comportamiento reutilizable sin pagar el coste completo del prompt en cada turno.

DeepSeek Harness usa la misma primitiva para que la guía de revisión específica del proyecto, de creación de plugins y de uso de herramientas viva junto al workspace o a la configuración de agent del usuario, en lugar de estar codificada a mano en el loop.

## Decisión

`@deepseek-ai/dsh-skill` es el registro puro de providers (`ctx.skills`), `@deepseek-ai/dsh-skill-filesystem` es el provider local de sistema de archivos entregado, y `@deepseek-ai/dsh-tool-skill` posee el catálogo durable de sesión y la herramienta de carga orientada al modelo. `dsh-agent-spine-demo` carga por defecto el registro, el provider local y el Consumer, de modo que las apps TUI, headless y ACP obtienen el mismo comportamiento mientras los providers embebidos o remotos aportan skills sin cambiar el registro ni el Consumer. Su configuración `skills` reenvía las ramas `registry`, `local` y `tool` a esos propietarios.

Los providers empaquetados dedicados pueden aportar skills inmutables sin descubrimiento de sistema de archivos. La CLI entregada declara `@deepseek-ai/dsh-skill-badge` desactivado por defecto; habilitar su fila de composición aporta las instrucciones oficiales de la insignia a través del mismo registro y Consumer ([decisión](2026-08-06-bundled-dsh-badge-skill.es.md)).

Los plugins provider se registran síncronamente durante `apply()`. La membresía de providers es estado directo propiedad de efectos: el registro y la disposición invalidan síncronamente los catálogos completados, y el descubrimiento lee el mapa vigente de providers bajo demanda en lugar de observar eventos de cambio del registro. Los catálogos de providers retornan candidatos clasificados desde llamadas `list()` esperadas, donde los providers remotos realizan inicialización, autenticación y descubrimiento honrando la señal de abort de la búsqueda. El registro valida cada candidato, resuelve las skills de mismo nombre por primero-gana según clasificación, orden de registro de provider y orden local del provider, y luego ordena los resúmenes por nombre de skill para consumidores deterministas. Solo cachea instantáneas de catálogo completadas y reintenta cuando una revisión de provider/runtime cambia durante el descubrimiento, de modo que una descarga no puede congelar una skill obsoleta e irresoluble en un catálogo de sesión. El `ctx.skills.register(...)` de runtime sigue siendo una conveniencia para skills embebidas en proceso y usa prioridad proyecto-sobre-usuario; `runtime` está reservado como nombre de provider propiedad del registro.

El provider local escanea en orden de clasificación primero-gana las raíces de proyecto sensibles al cwd, las raíces personalizadas y las raíces de usuario: `.dsh` del proyecto, `.agents` del proyecto, `customSkillDirs`, `.dsh` de usuario y luego `.agents` de usuario. El escaneo de `.dsh/skills` de usuario se salta `.system` para que un directorio propiedad del sistema no se trate como contenido de usuario normal. El provider local no sintetiza skills de sistema integradas; las raíces empaquetadas configuradas y los providers dedicados aportan skills adicionales.

Cada skill es o bien `<name>/SKILL.md` o bien `<name>.md` con frontmatter YAML. `name` y `description` son obligatorios; `whenToUse`, `metadata`, `disable-model-invocation` y `user-invocable` son opcionales. Los nombres son kebab-case. Los campos de invocación se proyectan en una política anidada tipada según lo definido por [la decisión de invocación independiente para modelo y usuario](2026-07-28-skill-invocation-policy.es.md); el parser rechaza las grafías antiguas en camel-case. El frontmatter YAML se parsea con el paquete `yaml` en lugar de `js-yaml` o un parser escrito a mano: `yaml` es el parser moderno ya declarado para las necesidades limitadas de frontmatter de este paquete, y un parser estrecho o bien rechazaría YAML válido que los usuarios esperan que funcione o crecería hasta un subconjunto de YAML sin revisión.

La E/S local de archivos de skills pasa por `ctx.fs` cuando hay un servicio de sistema de archivos cargado: la búsqueda de raíz de proyecto sondea `.git` con `resolve` y `stat`, el descubrimiento de raíces usa `listDir`, y las lecturas de skills usan `readText`. El sistema de archivos de Node sigue siendo un fallback para contextos mínimos que montan `dsh-skill-filesystem` sin el seam de fs. Las raíces ausentes, los archivos de skills ilegibles o malformados y los fallos transitorios de `list()` del provider degradan a advertir-y-omitir para que una fuente mala no haga fallar cada petición del agent; los candidatos malformados siguen fallando rápido porque son violaciones del contrato del provider.

`dsh-tool-skill` inyecta un único catálogo durable `<system-reminder>` de rol usuario como `user/message` con origen en el primer `agent/pre-step` de la sesión, y solo cuando la vista de herramientas de ese agent resuelve el registro exacto `skill` de este plugin. El catálogo contiene solo nombre y descripción de skill ordenados; excluye cuerpos, rutas, fuentes, providers y pistas de enrutamiento. Las descripciones se normalizan en espacios en blanco, se escapan como XML y quedan topeadas por `catalogDescriptionMaxLength`, cuyo default es `500` y su mínimo `3`. Los cuerpos completos de las skills jamás se incluyen en el catálogo. (El catálogo viajaba originalmente en el [punto de extensión de prefijo de sesión solo-petición](../../archived/feature/2026-07-07-session-prefix.md), archivado; la [decisión de mensaje con origen unificado](../architecture/2026-07-22-unified-send-and-coalesced-user-messages.es.md) lo trasladó al historial durable.)

El `list()` del registro retorna todos los resúmenes ganadores, mientras que los consumidores de modelo y de usuario aplican los predicados de invocación propiedad de la [decisión de política de invocación independiente](2026-07-28-skill-invocation-policy.es.md). La herramienta `skill({ name })` carga una skill invocable-por-modelo para el cwd vigente del agent y retorna un resultado de herramienta con `<skill_content name="...">`, `<skill_resources>` y `<skill_instructions>`. `resourceBase` aporta un directorio, URL o base opaca gestionada por el provider para scripts, referencias y assets referenciados explícitamente; los recursos se cargan solo bajo demanda, sin enumeración de directorios. Un nombre sin resolver reporta que la skill es desconocida o ya no está disponible; los nombres inválidos y las skills con `invocation.modelInvocable: false` conservan errores de herramienta distintos. El resultado de la herramienta es la vía de revelación visible para el modelo.

Las estructuras de datos y el contrato de catálogo/herramienta están documentados en [skills.md](../../../../docs/subsystems/skills.es.md), con las firmas de servicio en el [catálogo de servicios generado](../../../../docs/subsystems/skills.es.md#cordis-surface).

## Alternativas consideradas

**Inyectar los cuerpos completos de las skills en cada system prompt.** Rechazado porque destruye la revelación progresiva y hace que cada petición pague por instrucciones que pueden no aplicar.

**Exponer las skills solo como slash commands.** Rechazado porque la carga iniciada por el modelo es la capacidad central; el anuncio por comando humano no cambia el descubrimiento.

**Poner el escaneo local del sistema de archivos directamente dentro de `ctx.skills`.** Rechazado porque los coding agents, los web agents y los ecosistemas futuros de plugins necesitan fuentes de skills distintas. Un registro de providers refleja el seam de subagente: el registro posee la resolución de conflictos y los consumidores, mientras que las implementaciones poseen la carga.

**Usar una sección del system prompt.** Rechazado porque el system prompt renderizado es una única cadena, mientras que el catálogo es un mensaje `<system-reminder>` de rol usuario. El [punto de extensión de prefijo de sesión solo-petición](../../archived/feature/2026-07-07-session-prefix.md) (archivado) era el mecanismo original; tras que la decisión de mensaje con origen unificado lo eliminara, el catálogo pasó a ser una inyección con origen durable con la misma forma de mensaje.

**Materializar las skills integradas de creación DSH bajo `~/.dsh/skills/.system`.** Rechazado porque las skills empaquetadas no escriben en el home del usuario en el arranque, y los providers embebidos o remotos aportan las skills configuradas.

**Descubrir recursivamente los `**/SKILL.md` anidados.** Rechazado. Los archivos planos y los paquetes de directorio de un nivel cubren las raíces configuradas manteniendo el manejo de duplicados y el orden del catálogo fáciles de razonar.

**Parsear el frontmatter a mano.** Rechazado porque el schema aceptado incluye un objeto `metadata` abierto. Un parser estrecho o bien rechazaría YAML válido que los usuarios esperan que funcione o crecería hasta un subconjunto de YAML sin revisión.

## Consecuencias

La espina del núcleo del agent incluye un contribuyente de catálogo, un provider local y una herramienta orientada al modelo. El descubrimiento de skills es sensible al cwd, así que los llamadores que crean agents con distintos valores de cwd de sesión pueden observar por diseño distintas anulaciones de skills de proyecto.

El catálogo es determinista para un conjunto fijo de raíces y una revisión de registro de runtime. El provider local vigila las raíces configuradas e invalida los catálogos completados tras cambios de disco relevantes; el registro de runtime y la disposición de providers también los invalidan.

## Aplazado

Los contextos de skill por fork (`context: fork`), las declaraciones y pistas de parámetros (`arguments` y `argument-hint`) y las restricciones de herramienta por skill (`allowed-tools` y `disallowed-tools`) quedan fuera del contrato entregado. El registro, el provider local y la herramienta orientada al modelo no parsean, anuncian ni hacen cumplir estos campos. La invocación directa por el usuario se entregó como una funcionalidad de TUI sobre la política de invocación compartida y la primitiva `get()` de confianza; véase [el slash command de skills del TUI archivado](../../archived/feature/2026-07-21-tui-skill-slash-command.md).
