# Agent Note: Disolución de toolview — las filas de herramientas son slots con clave por vista

Status: implemented

[English](2026-07-23-toolview-dissolution.md) | [中文](2026-07-23-toolview-dissolution.zh.md) | Español

> Ámbito: por qué se retiró el anillo de herramientas independiente (ToolViewRegistry/ctx.toolviews/outlet) y qué lo sustituyó. La [nota de arquitectura del cliente web](2026-07-19-gui-web-client-architecture.es.md) lleva la narrativa del estado publicado que produjo esta decisión; el [estándar del sistema de slots](2026-07-22-slot-type-chain-implementation.es.md) es dueño del modelo de registro sobre el que corre todo ahora. La decisión posterior de [propiedad de la presentación de las herramientas del cliente](2026-08-08-client-tool-presentation-ownership.es.md) sustituye solo a la colocación por vista de esta nota: el dispatch por nombre de Tool sigue siendo un slot con clave en lugar de un registro paralelo.

## Problema

Tras disolverse el anillo de vistas en el sistema de slots, el cliente conservaba exactamente un modelo de registro paralelo: el anillo de herramientas — un registro con nombre (`ctx.toolviews`) con su propia gramática de register, sus propias semánticas de resolución (dispatch de predicados scoped-beats-global), su propio par subscribe/version, su propia caché de inject y su propio outlet de render con un límite de error privado. Cada uno de esos elementos era una segunda implementación de algo que la maquinaria de slots ya poseía, y toda funcionalidad futura (un asiento de store para borradores de fila, inyección i18n, identidad entre bundles) habría tenido que construirse dos veces o derivar. La única justificación honesta del anillo era que los nombres de herramientas son un conjunto abierto en runtime mientras que `SlotMap` es una tabla de declaración cerrada — un registro con clave por cadenas arbitrarias parecía estructuralmente necesario.

## Decisión

El anillo de herramientas ha desaparecido como infraestructura independiente: una fila de herramienta es un **slot hijo con clave que cada vista declara para sí misma**, y el cliente tiene exactamente un modelo de registro. La justificación anterior era hueca — el *espacio de claves* de un slot con clave ya es abierto en runtime (SlotMap declara slots, nunca claves; el `key: 'question'` del componedor de ask-user era el precedente), así que el conjunto abierto de nombres de Tool encaja nativamente en el dispatch de `entryKey`.

Esta decisión colocó originalmente `'conversation.chat.toolview'` bajo la entry de chat e hizo que el sitio de render del chat despachara cada fila. La decisión posterior de [propiedad de la presentación de las herramientas](2026-08-08-client-tool-presentation-ownership.es.md) mueve esa colocación a un asiento de Tool completo y da a `ui-tool` un slot hijo con clave `'tool.call.toolview'`. Esa decisión posterior cambia el dueño de la presentación, no la restricción central de esta decisión: el registro de Tool sigue usando la maquinaria ordinaria de slots con clave, con activación, sustitución, caché, aislamiento de errores, versionado y comportamiento de fallback propiedad del framework.

## Cambios semánticos aceptados

Cuatro deltas de comportamiento se aceptaron deliberadamente, no se pasaron por alto. La apariencia entre vistas era inicialmente un registro por vista; la nota posterior registra por qué la composición raíz/subcall justificó después un único dueño de presentación para toda la Tool. El doble registro con la misma clave es un throw ruidoso donde el registro dejaba que el último ganara silenciosamente por encima del anterior — una corrección de disciplina, no una pérdida. El dispatch de la dimensión de sesión, cuando una fila lo necesita, pertenece al interior del componente (el kit estándar ya lleva `useSessions`), no a los predicados del registro — hoy no hay un ejemplar publicado de variante por sesión. La sobrescritura de forma a nivel de registro por terceros (un registro con ámbito que sombrea a uno global) no tiene equivalente; una necesidad futura real se enruta por convenciones de nombres de clave o un pequeño resolver dentro del componente, nunca un registro paralelo resucitado.

## Alternativas consideradas

**Conservar el registro independiente (la forma original).** Rechazado: cada uno de sus ejes de dispatch multidimensional tiene una casa más correcta — la propiedad de la presentación pertenece a un slot hijo declarado explícitamente, y la dimensión de sesión pertenece al interior del componente, que ya tiene el kit estándar. Lo que queda es una segunda copia de la maquinaria de slots sin ninguna capacidad distintiva.

**Promover `renderToolView` al kit estándar y mover el registro al paquete de runtime.** Rechazado: la presentación de Tool es vocabulario de la UI del Cliente; elevarla al runtime filtraría la presentación a la capa de objetos de datos y seguiría dejando dos modelos de registro.

**Derivar las declaraciones de slots de los refCounts de suscripción** (declarar el slot implícitamente cuando el primer registrante se suscribe). Rechazado por el acoplamiento implícito y la complejidad de debounce; anotado como posible revisita solo si aparece una UI realmente multivista.

**Una fachada fina `registerToolView` sobre slots.register.** Diferida, no rechazada: tras la disolución, la fachada llevaría solo azúcar de tiempo de compilación (estrechamiento literal de nombres de slot, vocabulario tool→key, precomposición de props) con cero runtime. Según «aplicar en el límite de la operación» (una fachada no es un punto de aplicación), permanece sin construir; la composición de tipos útil viaja como el alias exportado de props de la vista de Tool. Una fachada posterior puede añadirse sin perturbar el registro directo si la ceremonia repetida de registro lo justifica.

## Consecuencias

El cliente tiene un modelo de registro; auditar quién renderiza las llamadas de Tool significa leer las llamadas de registro de slots, la misma auditoría que para cualquier otro slot. Los registrantes obtienen gratis el aislamiento de errores, la caché de inject y el asiento de store del framework — ninguna capacidad viaja dos veces. Los costes son los cambios semánticos aceptados anteriores, sobre todo el fallo ruidoso de clave duplicada y la ausencia de sobrescritura a nivel de registro por terceros. Los registrantes independientes nombran el slot tipado en `ctx.slots.inject`, así que la dependencia es explícita y sigue la sustitución de declaraciones sin una convención de orden de servicios.
