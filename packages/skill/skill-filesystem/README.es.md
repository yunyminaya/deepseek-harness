# @deepseek-ai/dsh-skill-filesystem

[English](README.md) | Español

Provider local de sistema de archivos para el registro `ctx.skills`.

Este paquete implementa una fuente de skills (destreza). Escanea las raíces de skill locales del proyecto, personalizadas y de usuario, analiza los archivos de skill `SKILL.md` o de Markdown plano, y registra el provider en `ctx.skills`. El registro permanece en `@deepseek-ai/dsh-skill`; los catálogos durables de sesión y la herramienta loader orientada al modelo permanecen en `@deepseek-ai/dsh-tool-skill`.

## Plugin

Requiere `ctx.skills` (`inject: ['skills']`).

### Config

| Campo | Predeterminado | Significado |
|---|---|---|
| `providerName` | `filesystem` | Nombre único usado para registrar este provider en `ctx.skills`. |
| `includeDefaultRoots` | `true` | Incluye las raíces de proyecto y de usuario alrededor de `customSkillDirs`; ponlo en false para un provider aislado con raíz personalizada. |
| `dshHome` | `$DSH_HOME` o `~/.dsh` | Raíz de configuración de DeepSeek Harness resuelta por [`@deepseek-ai/dsh-home-paths`](../../util/home-paths/README.es.md); escanea `skills` bajo este directorio. |
| `agentsHome` | `$DSH_AGENTS_HOME` o `~/.agents` | Raíz de configuración de agent compartida escaneada en busca de skills compatibles. |
| `customSkillDirs` | `[]` | Raíces de skill locales adicionales escaneadas después de las raíces de proyecto y antes de las de usuario. |
| `watch` | `true` | Vigila las raíces locales del host e invalida el provider local cuando la pertenencia al catálogo o el frontmatter pueden haber cambiado. |
| `watchUsePolling` | `false` | Usa el sondeo de Chokidar en lugar de los eventos nativos para las raíces de skill existentes. |
| `watchStabilityThresholdMs` | `200` | Ventana de escritura estable para los eventos `add` y `change` de Chokidar. |
| `watchPollIntervalMs` | `100` | Intervalo de sondeo/estabilidad de Chokidar e intervalo de sondeo de rutas ausentes. |
| `watchMaxProjects` | `128` | Número máximo de raíces de proyecto distintas retenidas en el LRU del watcher. |
| `watchFollowSymlinks` | `true` | Sigue los enlaces simbólicos al vigilar raíces existentes. |

## Descubrimiento

Las raíces predeterminadas se resuelven en el orden de rango de este provider:

| Rango | Origen | Ruta |
|---|---|---|
| 100 | `project-dsh` | `<projectRoot>/.dsh/skills` |
| 200 | `project-agents` | `<projectRoot>/.agents/skills` |
| 300 | `custom` | `Config.customSkillDirs` |
| 400 | `user-dsh` | `<dshHome>/skills` |
| 500 | `user-agents` | `<agentsHome>/skills` |

La raíz de proyecto es el ancestro más cercano que contiene `.git`; sin uno, se usa el cwd actual. La raíz DSH de usuario omite su hijo `.system` para que los directorios propiedad del sistema no se traten como skills normales de usuario. `includeDefaultRoots: false` omite las filas de proyecto y de usuario y el valor predeterminado de entorno `$DSH_BUNDLED_SKILL_DIR` al tiempo que conserva las raíces personalizadas y empaquetadas configuradas explícitamente, lo que permite que varios providers aislados con nombres únicos vean solo sus propias raíces. Este provider suministra los skills de proyecto y de usuario; otro provider puede suministrar los skills de sistema integrados.

Cuando `ctx.fs` está disponible, el descubrimiento lista las raíces a través de `ctx.fs.listDir`, lee los archivos de skill a través de `ctx.fs.readText` y sondea `.git` a través del servicio de sistema de archivos. Las cargas completas de skill reenvían la señal de aborto de la búsqueda a los metadatos del sistema de archivos y a las lecturas de contenido. Sin un servicio de sistema de archivos, el provider recurre a E/S de sistema de archivos de Node cancelable, de modo que los contextos locales mínimos aún pueden cargar skills. Las rutas confirmadas como ausentes son un estado vacío válido, las entradas malformadas o no textuales avisan y se omiten, y los fallos inesperados de descubrimiento/lectura dejan la instantánea del registro incompleta en lugar de reemplazar el último catálogo de modelo bueno con una eliminación engañosa.

## Detección de cambios en el catálogo

Las raíces de skill existentes se vigilan con Chokidar. Antes de abrir un watcher nativo, el provider calcula la ruta real (realpath) de la raíz existente o de su ancestro y restaura el siguiente segmento ausente; cuando `watchFollowSymlinks` es false y la propia raíz es un enlace simbólico, conserva ese enlace final para que Chokidar pueda aplicar el límite configurado. El descubrimiento y los diagnósticos conservan la ruta configurada, mientras que Windows no puede mezclar de otro modo un alias 8.3 con eventos libuv de formato largo. El provider observa las altas/bajas directas de directorios bundle, las altas/bajas de Markdown plano y las altas/bajas/cambios directos de `SKILL.md`; `change` existe para redescubrir frontmatter del catálogo como `name` y `description`. Los cambios por debajo de `references`, `scripts`, `assets` u otros recursos bundle no invalidan el catálogo. Los eventos entregados en el mismo lote de microtareas se colapsan en una sola invalidación del provider.

Una raíz que no existe se sigue desde el ancestro existente más cercano un segmento de ruta ausente a la vez. El siguiente segmento se sondea con `fs.watchFile`; en cuanto aparecen `.agents`, `skills` o la raíz configurada, la observación avanza hasta que Chokidar puede adjuntarse a la raíz real. La eliminación de la raíz invierte este proceso, de modo que borrar y recrear un directorio de skills completo sigue siendo observable. Los watchers con ámbito de proyecto están acotados por `watchMaxProjects`; volver a visitar un proyecto desalojado reactiva la observación durante el descubrimiento.

Las herramientas `write` y `edit` de sistema de archivos de primera parte también invalidan síncronamente el provider a través de `fs/observed` cuando su destino podría afectar a una entrada de skill vigilada. Esta vía rápida hace que el siguiente paso del modelo observe su propia mutación del sistema de archivos sin esperar al watcher del host. Los cambios externos de IDE, Git, shell y procesos dependen de Chokidar o de la sonda de rutas ausentes. Los watchers de raíces existentes permanecen persistentes hasta el teardown del efecto, de modo que Chokidar es dueño de los eventos de error nativos asíncronos; los fallos de watcher en el arranque/runtime se registran y reintentan. El descubrimiento sigue escaneando las raíces legibles y devuelve sus candidatos para la carga directa, pero marca la observación como incompleta para que no se cachee ni se publique como catálogo de modelo autoritativo. El teardown del efecto cierra todos los watchers y contiene las devoluciones de llamada tardías.

## Formato de skill

Los skills pueden ser bundles de directorio de un solo nivel (`<name>/SKILL.md`) o archivos Markdown planos (`<name>.md`). El descubrimiento anidado de `**/SKILL.md` se excluye deliberadamente. El frontmatter se analiza como un objeto YAML abierto con el paquete `yaml`; este provider interpreta los obligatorios `name` y `description`, más los opcionales `whenToUse`, `metadata`, `disable-model-invocation` y `user-invocable`. Los nombres deben estar en kebab-case.

Los dos campos de invocación aceptan booleanos YAML y las formas insensibles a mayúsculas `true`/`false`, `yes`/`no`, `on`/`off` y `1`/`0`. `disable-model-invocation: true` excluye el skill de los catálogos y loaders orientados al modelo; `user-invocable: false` lo excluye de los comandos orientados a humanos. Cada campo omitido se predetermina a permitir su superficie, y el provider emite siempre ambos valores positivos de política interna, incluso cuando faltan las dos claves. Una ortografía camel-case rechazada o un valor de invocación no booleano descarta el skill entero del descubrimiento con un aviso, en lugar de descartar solo ese campo o caer en un predeterminado permisivo. La política de invocación falla en fail-closed porque ignorar datos no válidos podría exponer un skill en una superficie deshabilitada; los valores opcionales mal tipados de `whenToUse` y `metadata` se omiten porque ninguno concede invocación actualmente.

El catálogo y el cuerpo tienen ciclos de vida separados. El descubrimiento analiza el frontmatter para producir el resumen. Cada carga de `skill(name)` relee y reanaliza el archivo actual, de modo que las ediciones del cuerpo no necesitan hash, revisión, invalidación de caché ni notificación proactiva al modelo. Un cambio de nombre en el frontmatter entre el descubrimiento y la carga rechaza el nombre obsoleto e invalida el provider; la siguiente observación del catálogo publica el nombre nuevo.

## Experiencia del modelo

De forma indirecta, a través de `dsh-tool-skill`, que renderiza los nombres invocables de este provider y sus descripciones con tope en el catálogo inicial o de reemplazo, y un cuerpo de instrucciones actual seleccionado más la guía de la base de recursos en el historial de herramientas retenido, mientras que las rutas, los rangos de provider y los skills deshabilitados permanecen ocultos.

#### Efecto en la caché KV

La invalidación del watcher puede hacer que el Consumer nombrado añada un catálogo de reemplazo al historial de solicitudes existente. Las ediciones solo de cuerpo dejan el resumen del catálogo sin cambios.

## Limitaciones conocidas y trabajo diferido

- **El descubrimiento tiene un solo nivel de profundidad** — solo se reconocen `<root>/<name>/SKILL.md` y `<root>/<name>.md`; los árboles de skill anidados y los manifests de paquete se ignoran.
- **El ámbito de proyecto es el ancestro `.git` más cercano** — los espacios de trabajo sin ese marcador recurren al cwd suministrado, sin marcador alternativo de raíz de proyecto ni selección de subproyecto en monorepos.
- **Las entradas malformadas desaparecen con un aviso** — el catálogo del modelo no recibe ningún diagnóstico por skill y no puede distinguir un skill ausente de uno no válido; los fallos inesperados de E/S conservan en cambio el último catálogo bueno.
- **La observación de raíces ausentes sondea un segmento de ruta** — las raíces ausentes al arrancar usan `fs.watchFile` a `watchPollIntervalMs` hasta que Chokidar puede adjuntarse, cambiando latencia de detección acotada por detección fiable de creación en flujos de trabajo de IDE, Git y shell.
- **Sin protocolo de revisión de cuerpo** — un cuerpo cargado es historial de herramientas retenido ordinario; las ediciones posteriores del archivo afectan a llamadas posteriores, pero ni reescriben resultados antiguos ni anuncian que el cuerpo cambió.
