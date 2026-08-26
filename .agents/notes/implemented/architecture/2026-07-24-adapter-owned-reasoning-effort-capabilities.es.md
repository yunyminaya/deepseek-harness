# Agent Note: Capacidades de esfuerzo de razonamiento propiedad del adaptador

Status: implemented

[English](2026-07-24-adapter-owned-reasoning-effort-capabilities.md) | [中文](2026-07-24-adapter-owned-reasoning-effort-capabilities.zh.md) | Español

## Problema

La fuerza de razonamiento era solo configuración del adaptador, así que una conversación no podía descubrir ni cambiar entre solicitudes los niveles soportados del modelo seleccionado. Promover la unión de niveles de un adaptador a `dsh-llm` haría que cada provider y modelo adoptara nombres que quizá no soporta, mientras que una bolsa de opciones específica del provider impediría al loop validar o reconstruir durablemente la solicitud efectiva.

## Decisión

`dsh-llm` representa un esfuerzo de razonamiento como el `ReasoningEffortId` opaco y con marca. Una consulta `resolveModel(provider, model, signal?)` propiedad del adaptador devuelve `LlmResolvedModelInfo`: la identidad exacta del modelo más metadatos opcionales de contexto y de razonamiento. `LlmRuntime.resolveModelInfo()` valida y separa ese agregado. Cuando está presente, `reasoning.efforts` es una lista ordenada no vacía de ids con metadatos de visualización y puede nombrar un default configurado. El core exige que un esfuerzo explícito o configurado aparezca exactamente en esa lista y nunca hace clamp ni alias de un valor.

`LlmCallConfig` y `GenerateOptions` llevan el esfuerzo opcional. El agent loop prepara la config posterior a `agent/request` bajo la señal del turno activo antes de escribir `request/header`, así que los defaults y los cambios dinámicos solo son visibles para el modelo después de convertirse en hechos durables. La llamada preparada conserva el registro exacto del adaptador a través de la resolución asíncrona del modelo exacto, el registro durable del header y el dispatch; las llamadas directas a `LlmRuntime.stream()` capturan igualmente su registro final antes de esperar la resolución. Una ruta sin adaptador registrado conserva su config propuesta para que un middleware `llm/stream` pueda hacerse dueño de ella y cortocircuitarla; el dispatch terminal sigue rechazando una ruta no gestionada. Un loop reanudado conserva el esfuerzo registrado solo cuando su ruta inicial de provider/modelo no cambia; un cambio de ruta descarta el id opaco del modelo anterior.

El adaptador nativo de DeepSeek anuncia `off`, `low`, `high` y `max` cuando la política de despliegue permite el pensamiento, y usa por defecto el esfuerzo configurado o `high`. Su `off` propiedad del adaptador mapea a `thinking.type: disabled` sin `reasoning_effort`; `low`, `high` y `max` activan el pensamiento y llevan su esfuerzo oficial de cable homónimo. Un despliegue `thinking: disabled` publica solo `off` y rechaza los intentos de activar el pensamiento antes de la E/S del provider. El adaptador pi-ai publica el resultado de `getSupportedThinkingLevels()` de cada modelo exacto sin cambios, incluido `off`, conserva un default de perfil ausente como default del provider y deja el mapeo de valores de cable del provider dentro de pi-ai. Sus opciones de stream comunes representan `off` omitiendo `reasoning`, como exige la propia API de pi-ai.

## Alternativas consideradas

**Definir la unión `ThinkingLevel` de pi-ai en el core.** Rechazado porque los nombres canónicos actuales de pi-ai son un detalle de implementación del adaptador; un provider futuro puede exponer un identificador distinto sin requerir una release del core.

**Llevar un objeto de opciones de provider sin tipos.** Rechazado porque el loop no podía ni validar un valor seleccionado ni poner un hecho estable neutral al provider en el header de la solicitud.

**Hacer clamp de los niveles no soportados.** Rechazado porque una sustitución silenciosa hace que el control seleccionado por el usuario difiera de la intención de la solicitud registrada y oculta la configuración de despliegue obsoleta.

**Normalizar cada adaptador a una lista de niveles propiedad del core o eliminar `off`.** Rechazado porque el vocabulario seleccionable pertenece a la capacidad exacta del modelo. Un cliente puede renderizar la opción `off` de un adaptador sin exigir que todo adaptador la exponga.

## Consecuencias

Los clientes pueden consultar una ruta exacta una vez y renderizar su identidad, su capacidad de contexto y sus opciones de razonamiento propiedad del adaptador sin conocer un enum global ni sintetizar `off`. La configuración del adaptador sigue siendo el dueño del default de despliegue y de la política, mientras que `agent/request` puede sustituir el esfuerzo efectivo en cada paso dentro de esa política. La identidad, el contexto o los metadatos de razonamiento exactos no válidos fallan con `INVALID_MODEL_INFO`, `INVALID_MODEL_CONTEXT` o `INVALID_MODEL_REASONING`; los valores explícitos o configurados no soportados fallan con `UNSUPPORTED_REASONING_EFFORT` antes de la E/S del provider.

La consulta agregada del modelo exacto es asíncrona y puede fallar para adaptadores respaldados por catálogos autoritativos. Su señal opcional es el límite de cancelación del llamador; un adaptador asíncrono debe asentarse con prontitud tras el abort para que el dispose del loop pueda alcanzar la quietud. Las pruebas sin clave de servicio, adaptador, loop, sesión y header de solicitud fijan la validación, los defaults, los cambios dinámicos, el registro, el comportamiento de reanudación, la propiedad del registro HMR y la cancelación; las instantáneas ejecutables fijan el esfuerzo resuelto en headers de solicitud reales ensamblados, mientras que las pruebas de adaptador con clave ejercitan la serialización del provider.
