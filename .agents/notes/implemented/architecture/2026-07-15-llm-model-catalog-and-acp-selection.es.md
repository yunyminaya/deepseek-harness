# Agent Note: Catálogos LLM de asesoramiento y selección de modelo ACP por sesión

Status: implemented

[English](2026-07-15-llm-model-catalog-and-acp-selection.md) | [中文](2026-07-15-llm-model-catalog-and-acp-selection.zh.md) | Español

> La decisión del catálogo sigue vigente. La selección de modelo ACP por sesión queda sustituida por [ACP como protocolo solo de automatización](../simplification/2026-07-23-acp-automation-only-protocol.es.md).

## Problema

Los adaptadores enrutados por provider permiten que cada petición elija `provider + model`, pero `LlmRuntime` solo exponía el enrutamiento y el streaming. Una UI no podía descubrir qué providers estaban registrados ni qué modelos estaba dispuesto a recomendar un adaptador. Los clientes ACP, por tanto, no recibían ninguna opción de configuración `model` de sesión, así que las integraciones con Zed, JetBrains y VS Code no tenían lista de modelos aunque el servicio LLM ya soportara el cambio de modelo en caliente.

El descubrimiento de modelos no puede convertirse en validación de peticiones. El adaptador DeepSeek escrito a mano reenvía deliberadamente ids de modelo arbitrarios a un endpoint público o privado, mientras que pi-ai tiene un catálogo instalado finito que es autoritativo para su propia resolución de peticiones. Tratar un catálogo compartido como lista blanca eliminaría el comportamiento de endpoint privado que el enrutamiento por provider fue diseñado para preservar.

La selección ACP también debe preservar la dimensión del provider. El mismo id de modelo puede aparecer bajo varias rutas, y cambiar un adaptador global o una plantilla de agent filtraría la elección de una sesión de editor a todas las demás. Las variables de prompt y el enrutamiento de peticiones deben cambiar a la vez; una selección que se decide durante el ensamblado asíncrono del prompt no puede hacer que `{{model}}` nombre un modelo mientras la petición llega a otro.

## Decisión

### Descubrimiento de asesoramiento neutral al provider

`LlmAdapter` gana los métodos `providerInfo(provider)` y `listModels(provider)` asíncrono. Sus resultados neutrales al provider son `LlmProviderInfo { id, name }` y `LlmModelInfo { provider, id, name, description? }`. Los valores por defecto preservan el comportamiento existente de los adaptadores nombrando un provider por su ruta y no anunciando ningún modelo.

`LlmRuntime.listProviders()` devuelve metadatos desacoplados en orden de registro. `LlmRuntime.listModels(provider)` delega en el propietario de la ruta, valida que los ids y los nombres no estén vacíos, rechaza un provider no coincidente o un id de modelo duplicado con `INVALID_CATALOG` y devuelve valores desacoplados. Los providers desconocidos siguen fallando con `NO_ADAPTER`. Los metadatos de provider se validan atómicamente durante `registerAdapter()`, de modo que un registro de visualización malformado no pueda dejar un registro a medias.

La pertenencia al catálogo es de asesoramiento. Impulsa selectores y diagnósticos, pero nunca cambia el enrutamiento de `stream()` ni rechaza una petición por lo demás válida. La propiedad del provider sigue siendo exclusiva y ligada al ciclo de vida; los ids de modelo siguen siendo entrada de adaptador en tiempo de petición.

`dsh-llm-pi-ai` proyecta las entradas instaladas de `getModels(provider)` del provider configurado en el catálogo neutral. Su búsqueda de catálogo existente en tiempo de petición sigue siendo autoritativa y continúa rechazando modelos desconocidos con `UNKNOWN_MODEL`. `dsh-llm-deepseek` acepta una configuración `models` opcional que contiene entradas de visualización, con valores por defecto `deepseek-v4-flash` llamado `DeepSeek-V4-Flash`, `deepseek-v4-pro` llamado `DeepSeek-V4-Pro` y `deepseek-v4-flash-vision-exp` con capacidad de imagen llamado `DeepSeek-V4-Flash-Vision-Exp`. Una lista explícita reemplaza esos valores por defecto y una lista vacía desactiva el descubrimiento. Las entradas mejoran la UX del selector para modelos públicos o privados conocidos, mientras que todo id de modelo no listado sigue pasando sin cambios.

### Selección por sesión en el front end

Una selección la posee el front end que la ofrece (hoy el selector `/model` de la TUI), nunca `LlmRuntime` ni `AgentOptions`: esos son objetos de ámbito de despliegue o de creación, y mutarlos acoplaría sesiones concurrentes. Cada elección opaca lleva el par completo provider/modelo, porque el mismo id de modelo puede aparecer bajo varias rutas.

El transporte de automatización ACP no consume el catálogo. Su configuración de despliegue suministra un objetivo opcional provider/modelo para los agents recién creados, y no anuncia ningún selector de modelo ni interfaz de opciones de configuración.

### Coherencia y durabilidad de prompt/petición

`installModelSelection` (en `dsh-agent`) instala listeners con ámbito `system-prompt/assemble` y `agent/request` para una selección propiedad del front end. El ensamblado de prompt captura una instantánea del par seleccionado una vez por paso, sobrescribe las variables `provider` y `model` ensambladas después de los listeners de prompt posteriores, y el listener de petición aplica esa misma instantánea después de los listeners de petición posteriores. Una selección que se decide durante el ensamblado asíncrono empieza por tanto en el siguiente paso, en lugar de separar el texto del prompt del enrutamiento. El resto de campos de configuración de llamada permanecen intactos.

La cabecera de petición sigue siendo la fuente de verdad durable. Cuando una selección se usa de hecho, la instantánea completa existente `request/header` la registra, y un front end inicializa su selección desde la última cabecera de petición plegada antes de recurrir a las opciones de creación. Una selección que ninguna petición llega a usar es intencionadamente solo de memoria, porque nunca se convirtió en estado visible para el modelo.

## Alternativas consideradas

**Devolver solo cadenas de modelo.** Un valor solo de modelo pierde la ruta del provider y se vuelve ambiguo en cuanto dos providers exponen el mismo id.

**Convertir los catálogos en listas blancas obligatorias.** Esto choca con el paso directo de modelos arbitrarios del adaptador escrito a mano y con los despliegues privados. El adaptador seleccionado ya posee la validación autoritativa de peticiones.

**Guardar la selección en `AgentOptions` o `LlmRuntime`.** Esos son objetos de ámbito de creación o de despliegue. Mutarlos acoplaría sesiones concurrentes y eludiría la ruta registrada de reemplazo `agent/request`.

**Persistir inmediatamente un nuevo evento de sesión de selección de modelo.** Una selección de UI no usada no ha afectado a ninguna petición de modelo. Registrar la cabecera de petición existente cuando el objetivo se consume preserva la regla de «visible para el modelo si y solo si se registra» sin añadir una segunda fuente de verdad.

## Consecuencias

- Cualquier adaptador puede exponer una lista de modelos dinámica sin filtrar tipos de la biblioteca del provider en la Service Definition de LLM.
- Los consumidores de catálogo deben tratar la ausencia como «no anunciado», nunca como «petición no válida».
- Los adaptadores pi-ai exponen sus catálogos de providers instalados; los despliegues DeepSeek escritos a mano listan explícitamente las opciones conocidas y conservan el soporte de modelos arbitrarios.
- Los consumidores de catálogo orientados a humanos son dueños de su interacción de selección. ACP usa su objetivo de despliegue fijo y no amplía el protocolo con descubrimiento de modelos.
- Las cabeceras de petición siguen siendo compatibles con la forma de sesión enrutada por provider; no se requiere ningún evento JSONL nuevo ni versión de formato.
- Una lectura de catálogo puede ser asíncrona y todo llamador recibe valores desacoplados.

## Pruebas

La cobertura unitaria valida el desacoplamiento del catálogo y los metadatos malformados, la proyección de catálogo de pi-ai y DeepSeek, el enrutamiento de peticiones provider/modelo y la alineación de las variables de prompt; el aislamiento por agent se sigue de instalar los listeners en el contexto con ámbito del agent. Las pruebas de transporte ACP validan el reenvío fijo provider/modelo con independencia del descubrimiento de catálogo; la suite de TUI cubre la interacción del selector y la restauración basada en cabecera.
