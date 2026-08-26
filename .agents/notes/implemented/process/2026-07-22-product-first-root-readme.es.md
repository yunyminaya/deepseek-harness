# Agent Note: Product-first root README

Status: implemented

English | [中文](2026-07-22-product-first-root-readme.zh.md) | Español

## Problema

El README raíz es el punto de entrada de producto del repositorio. Su estructura de producto primero y su voz asentada siguen siendo útiles, pero los puntos de entrada concretos y las afirmaciones de capacidad se desfasan a medida que el runtime crece. Reescribir secciones cuyos hechos siguen siendo correctos aumenta la superficie de revisión y descarta redacción que ya funciona.

## Decisión

El README raíz preserva su estructura, su orden y su redacción existentes allá donde el hecho subyacente sigue siendo correcto. Una renovación cambia solo las afirmaciones desfasadas y añade el material necesario para representar las superficies enviadas; no usa el crecimiento del repositorio como motivo para replantear la página entera.

Una nota antes de la instalación agradece a los probadores internos, declara que las funcionalidades y la experiencia siguen inacabadas y pide reportes directos de fallos, confusión y fricción por el grupo de WeCom. La declaración de etapa de desarrollo existente identifica DeepSeek Harness como estando en pruebas internas.

La sección de superficie de usuario añade el servidor de automatización ACP y los SDK de Python/JSON-RPC junto a las entradas existentes de Web, TUI y headless. La TUI instalada sigue siendo el único comando `dsh`; las instrucciones de Web compilan el checkout activo antes de ejecutar `dsh web`, y las rutas de checkout personalizadas o reutilizadas se mantienen explícitas. Estos caminos de lanzamiento deben seguir siendo ejecutables a través de un PTY real y de una prueba de humo de build/HTTP de producción, respectivamente. El párrafo de capacidades mantiene su estilo compacto de inventario al tiempo que añade las familias enviadas de PTY, LSP, web, objetivos, planificación, tareas, sandbox, aprobación, ajustes, credenciales, session-query y telemetría, y declara que las composiciones seleccionan subconjuntos. Una viñeta adyacente registra la regla del registro de sesión autoritativo porque la persistencia, la reproducción, las consultas, la telemetría y las interfaces dependen de ella.

Los inventarios detallados de paquetes y servicios siguen en su documentación propietaria. Los lados inglés y chino del README comparten la misma estructura técnica, mientras que sus secciones de comunidad siguen apuntando al canal primario de cada audiencia lingüística. El sitio web de documentación mantiene una [ruta de entrada de inicio rápido separada](../simplification/2026-08-11-quickstart-documentation-home.es.md) en lugar de presentar otra página de aterrizaje de producto.

## Alternativas consideradas

**Reescribir el README en torno a una nueva narrativa de producto.** Una reescritura completa puede dar prominencia a todas las superficies actuales, pero sustituye texto exacto ya revisado y crea agitación innecesaria. Los hechos vigentes caben en la estructura de producto primero ya asentada.

**Presentar el repositorio como un SDK y un catálogo de paquetes.** Esto expone la amplitud de la implementación de inmediato, pero obliga a un lector nuevo a reconstruir el producto a partir de nombres de paquetes. El mapa de paquetes y el grafo de capacidades generado siguen siendo los inventarios autoritativos.

**Usar una larga página de marketing con capturas, insignias y tutoriales duplicados.** Los medios ricos pueden mostrar un viaje de producto estable, pero envejecen por separado respecto de los comandos y los contratos de código fuente. La raíz se mantiene compacta y enlaza a los ejemplos ejecutables y a las guías propietarias.

**Proyectar el README raíz como página de inicio del sitio web de documentación.** Una única página de aterrizaje evita dos narrativas, pero la guía de usuario del sitio web y el punto de entrada de producto/desarrollador del repositorio tienen necesidades distintas de navegación y mantenimiento. La raíz de la documentación envía a los lectores al inicio rápido.

## Consecuencias

Los revisores pueden distinguir las renovaciones de hechos de las reescrituras editoriales, y las actualizaciones futuras conservan la redacción asentada salvo que su significado se vuelva falso o incompleto. El README debe seguir cambiando con los comandos, puntos de entrada, afirmaciones de etapa de lanzamiento o familias de capacidades de alto nivel afectados, mientras que el detalle exhaustivo sigue enlazado en lugar de copiado.
