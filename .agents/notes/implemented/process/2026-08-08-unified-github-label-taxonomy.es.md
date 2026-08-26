# Agent Note: Taxonomía unificada de etiquetas de GitHub

Status: implemented

[English](2026-08-08-unified-github-label-taxonomy.md) | [中文](2026-08-08-unified-github-label-taxonomy.zh.md) | Español

## Problema

Las etiquetas de los pull requests responden a dos preguntas independientes: qué tipo de cambio introduce el trabajo y qué dominios duraderos del repositorio afecta de forma material. Mezclar esas dos dimensiones o conservar sinónimos entre etiquetas simples y etiquetas con espacio de nombres vuelve ambiguas las consultas, mientras que un inventario cerrado de áreas obliga a meter los dominios nuevos en categorías inexactas.

Las incidencias ya tienen un Issue Type nativo y una taxonomía de origen separada. Reutilizar las etiquetas de tipo u origen de los pull requests en ambos tipos de objeto duplica metadatos y debilita el significado de cada familia.

## Decisión

Todo pull request abierto o fusionado lleva exactamente una etiqueta canónica `kind/*` y al menos una etiqueta `area/*` que afecte de forma material. Los pull requests cerrados que nunca se fusionaron conservan las asignaciones históricas migradas, pero no reciben clasificaciones inventadas para lo que falta. Las etiquetas operativas pueden coexistir sin satisfacer ninguna de las dos dimensiones.

### Tipos

El conjunto de tipos es cerrado y mutuamente excluyente:

| Tipo | Significado |
|---|---|
| `kind/feature` | Añade o cambia intencionadamente el comportamiento. |
| `kind/bug-fix` | Corrige un comportamiento incorrecto. |
| `kind/doc` | Hace que la documentación sea la intención dominante. |
| `kind/testing` | Cambia pruebas o infraestructura de pruebas sin cambiar el comportamiento del producto. |
| `kind/cleanup` | Conserva el comportamiento mientras mantiene o simplifica la implementación o el proceso del repositorio. |
| `kind/dependency` | Actualiza dependencias sin otra intención dominante. |

El tipo registra la intención dominante. Las pruebas, la documentación, la limpieza o los movimientos de dependencias que acompañan al cambio no anulan una funcionalidad ni una corrección de errores. Un tipo nuevo cambia estas reglas de clasificación y exige un cambio explícito de taxonomía y de política.

La política del repositorio rechaza los valores `kind/*` no admitidos y reserva todos los alias eliminados por la unificación: `kind/bug`, `kind/documentation`, `feature`, `bug-fix`, `doc`, `cleanup`, `testing`, `dependencies`, `ci`, `cli`, `llm` y `web-search`. Reservar exactamente el conjunto migrado impide que un sinónimo obsoleto se recree como una etiqueta operativa aparentemente no relacionada.

### Áreas

Las áreas nombran asuntos duraderos del producto o de la ingeniería, no iniciativas temporales, titularidades ni todas las rutas tocadas de pasada. Un pull request lleva varias áreas cuando cambia comportamientos o APIs distintos, pero no combina un paraguas y una etiqueta más estrecha para el mismo cambio. Los nombres y descripciones en vivo de `area/*` en GitHub son los dueños del inventario actual; este registro define los casos de selección que no caben de forma fiable en descripciones cortas de etiquetas.

- `area/web` cubre las interfaces gráficas de navegador y Electron, `area/vscode` cubre la extensión del editor y `area/api` cubre los protocolos entre interfaces y los SDK de lenguaje.
- `area/planning` cubre objetivos, planes, tareas pendientes y planificación, mientras que `area/workflow` cubre los flujos de trabajo ejecutables y los runtimes de trabajos en segundo plano.
- `area/artifact` combina deliberadamente artefactos, adjuntos y entrega multimodal. Las etiquetas separadas solo se justifican cuando esas preocupaciones vuelvan a necesitar revisión o consultas independientes.
- `area/tools` se aplica a contratos genéricos de registro, schema y ejecución. Una capacidad concreta usa su propia área salvo que también cambie uno de esos contratos.
- `area/hooks` significa los puentes de Claude Code y Codex, `area/infra` cubre compilación, lanzamientos, CI, puertas del repositorio, generadores, dependencias y herramientas de desarrollo, y `area/windows` cubre el soporte nativo del producto en Windows, no la selección del runner de CI.

El conjunto de áreas es extensible a propósito. Cuando ninguna descripción existente cubre con honestidad un dominio duradero y reutilizable, un agente puede crear una etiqueta concisa `area/<lowercase-kebab-case>` sin aprobación separada. No debe crear un área para un único pull request, una ruta incidental, un proyecto temporal, un estado o una persona o equipo, y comunica la nueva etiqueta y su justificación a quien la solicitó después de aplicarla. Reutilizar un área inexacta solo para evitar una incorporación justificada no es aceptable.

### Incidencias y migraciones

Las incidencias usan el Issue Type nativo en lugar de `kind/*`; sus etiquetas `area/*` siguen siendo opcionales. Las etiquetas `source/*` registran cómo se creó una incidencia y no se aplican a los pull requests. La prioridad, los valores por defecto de GitHub y los disparadores de flujos de trabajo siguen siendo metadatos operativos independientes.

Las migraciones de etiquetas conservan el significado antes de eliminar alias: añade el reemplazo canónico, verifica lo etiquetable y elimina después la asignación obsoleta. Una etiqueta solo se borra cuando ningún pull request ni incidencia la usa todavía, y las etiquetas no relacionadas nunca se reemplazan en bloque.

## Alternativas consideradas

**Etiquetas sin prefijo.** Los nombres simples reducen el ruido visual, pero no identifican si una etiqueta clasifica intención, dominio, origen, prioridad o automatización. Conservar a la vez los sinónimos simples y los de espacio de nombres también vuelve ambiguas las consultas y la aplicación de la política.

**Un único conjunto de etiquetas sin diferenciar.** La presencia de una etiqueta no demostraría que se consideraron tanto la intención como el alcance semántico.

**Una lista fija de áreas en la política del repositorio.** Los dominios duraderos del repositorio evolucionan. El espacio de nombres `area/*` sigue siendo mecánicamente reconocible mientras las descripciones en vivo llevan el inventario extensible.

**Áreas derivadas de paquetes o rutas.** Las áreas describen el impacto semántico a través de los límites de los paquetes, mientras que las rutas cambiadas incluyen pruebas incidentales, documentación y archivos de soporte.

**Etiquetas separadas para cada shell de entrega o ciclo de vida de medios.** La entrega en navegador y Electron comparte un dominio gráfico, y la entrega de artefactos, adjuntos y multimodal comparte hoy un dominio de revisión y consulta. Una división pertenece a un cambio de taxonomía posterior solo cuando restaura una clasificación independiente útil.

**Etiquetas amplias de implementación en lugar de asuntos de producto o ingeniería.** Una capacidad concreta no es meramente su herramienta, interfaz, sistema de archivos o implementación de proceso. Las áreas genéricas de implementación se aplican solo cuando cambia su propio comportamiento o API.

**Tipos en las incidencias.** El Issue Type nativo ya es dueño de esa clasificación; duplicarla como etiqueta crea deriva.

**Exactamente un área por pull request.** Los cambios coherentes pueden afectar de forma material a varias APIs o comportamientos independientes, y descartar las áreas secundarias oculta el alcance afectado.

## Consecuencias

Los revisores y la automatización pueden consultar de forma independiente la intención, el alcance semántico, cómo se creó una incidencia, la prioridad y los disparadores operativos. Los mantenedores deben leer el cambio y las descripciones en vivo de las etiquetas en lugar de inferir la clasificación de los prefijos del título o de las rutas. El catálogo en vivo, esta justificación y la aplicación de la política deben moverse juntos cuando cambia un tipo o un límite de área no evidente, y las migraciones de taxonomía llevan un coste explícito de rellenado histórico y de verificación.
