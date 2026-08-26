# Agent Note: headless es un punto de entrada directo al núcleo

Status: implemented

[English](2026-08-09-headless-direct-core-entry-point.md) | Español

## Problema

El contrato de producto de `headless` es una tarea local con el texto final del asistente en stdout, un código de salida sensible al éxito, stderr vacío en caso de éxito y ningún puerto a la escucha. Una composición que contenga servicios de Workspace Host, ApiProxy, HTTP, el runtime Web o plugins de navegador contradice ese contrato y hace que la finalización local dependa de un árbol de transporte no relacionado.

El punto de entrada directo necesita igualmente el mismo estado del modelo de despliegue que los Agent creados por Web. Un default separado de provider/modelo daría a un mismo despliegue dos respuestas, mientras que derivar la finalización antes de que el Agent y la persistencia de Session estén en reposo permite que stdout y el código de salida observen estado incompleto.

## Decisión

El perfil `headless` enviado contiene `dsh-base` y `dsh-headless`. El bundle headless aporta su persona y su modo de herramienta, desactiva HMR, monta explícitamente el worker de Code Mode e inserta `headless-runner`. Su árbol no contiene ningún paquete `@deepseek-ai/dsh-host-*`, ni ApiProxy, ni servidor HTTP, ni runtime Web ni cliente de navegador. Code Mode y la persistencia de Session son capacidades de Agent de un solo disparo independientes de la presentación Web.

`headless-runner` es un punto de entrada directo al núcleo. Tras el asentamiento del Loader, lee `ctx.agentDefaultModel.currentSelection()`, crea un Agent persistido nuevo mediante `ctx.agents.create`, instala esa `ModelSelection` en el ámbito del Agent, espera la quietud del arranque, ancla la secuencia de Session, envía un mensaje de usuario ordinario y vuelve a esperar quietud. Espera a `ctx.sessions.flush`, pliega su intervalo de eventos durables para el último texto de asistente no vacío y la razón final de `turn/end`, escribe el texto más un salto de línea en stdout y solicita un apagado acotado del launcher con salida 0 exactamente cuando la razón es `completed`. Una razón de `error` terminal escribe su código y mensaje durables en stderr; los fallos inesperados del driver también usan stderr y salida 1.

`@deepseek-ai/dsh-agent-default-model` posee el default independiente del transporte usado para un Agent sin selección local de sesión. `AgentDefaultModelConfig` proporciona `ctx.agentDefaultModel` y registra la sección de Ajustes `agent-default-model`. La configuración de composición aporta `{provider, model}`; los ajustes de usuario también pueden aportar `reasoningEffort`. `currentSelection()` devuelve la selección completa viva y `saveSelection()` la escribe como sección completa, de modo que una selección sin effort limpia cualquier effort almacenado. `dsh-base` aporta la entrada de composición. Los puntos de entrada directo y de ApiProxy consumen este servicio; solo ApiProxy posee la precedencia local de sesión, la validación de modelos y la persistencia de las selecciones Web aceptadas.

`loadProfile` reconoce la tupla headless exacta propiedad de la instalación (`dsh-base`, `dsh-web-app`, `dsh-headless`) y la normaliza a la plantilla headless enviada conservando todos los demás campos del manifest. Las listas de bundles extra, ausentes o reordenadas son propiedad del usuario y permanecen intactas.

Esta nota posee los contratos de transporte y finalización headless. [Las apps son dueñas de sus líneas de comando](2026-08-06-app-owned-command-line.es.md) posee la gramática actual de `dsh --profile headless`; la antigua [decisión de `dsh run`](../../archived/feature/2026-08-08-dsh-run-headless-command.md) registra la gramática obsoleta propiedad del launcher, [el entramado de GUI y el protocolo RPC](2026-07-19-gui-layering-and-rpc-protocol.es.md) posee los límites del gateway de navegador, [el arranque del árbol de configuración web y el entramado de transporte](2026-07-24-web-config-tree-boot-and-transport-layering.es.md) posee el árbol Web, y [el modelo por defecto sigue al selector](../feature/2026-08-07-default-model-follows-the-picker.es.md) posee la persistencia del default compartido de Agent.

## Verificación

Los tests de paquete usan el store real de Session y el registro de Agent alrededor de una fábrica de Agent guionizada para fijar la agregación de reposo a reposo, la finalización asíncrona tardía, los diagnósticos terminales de modelo, otras salidas no completadas, los fallos directos, la disposición en tiempo de Loader y el orden de flush-antes-de-salir. Las instantáneas ensambladas sin clave conducen `dsh --profile headless` a través de un viaje de ida y vuelta de herramienta reproducido, registran un `user/message` con `source.kind: 'user'` y exponen un fallo terminal de modelo en stderr. La aceptación del binario compilado alcanza un provider mock a través del punto de entrada publicado y exige texto final en stdout, salida 0 y stderr vacío. La aceptación de volcado de configuración excluye todo paquete de Host, Web y Cliente del árbol headless enviado; la cobertura del apagado PTY no exige línea de observación y sí disposición acotada.

## Alternativas consideradas

| Alternativa | Desajuste de contrato |
|---|---|
| Conservar `dsh-web-app` pero suprimir su línea de observación | El proceso sigue abriendo un puerto y arrastra los árboles de Host, Web y navegador. |
| Construir un bundle de un solo disparo solo de Host alrededor de ApiProxy | ApiProxy es un gateway de protocolo de cliente; un punto de entrada local de un solo disparo no tiene límite de cliente. |
| Usar `InProcessApiClient` para cobertura de protocolo a nivel de producto | La ejecución del producto dependería de un protocolo no relacionado solo para ejercitar ese protocolo. |
| Dar a headless una configuración separada de provider/modelo | La creación directa y la Web tendrían defaults y persistencia independientes. |
| Omitir Code Mode y la persistencia de Session | Ambas capacidades pertenecen a la ejecución de un solo disparo del Agent, no a la presentación Web. |
| Normalizar toda tupla que contenga bundles Web y headless | Las listas de bundles son una superficie de extensión; solo la tupla exacta propiedad de la instalación es segura de clasificar. |

## Consecuencias

`dsh --profile headless` proporciona una tarea local de Agent en lugar de observación de navegador, APIs de Host o HTTP. Los usuarios que necesiten esas capacidades eligen `dsh web`. En caso de éxito, stderr está vacío, la finalización sigue al flush durable y la Session persistida permanece disponible para herramientas posteriores. Su mensaje de usuario inicial registra `source.kind: 'user'` y por tanto no lleva `rpcId` de ApiProxy.

La cobertura del portador ApiProxy sigue en el paquete ApiProxy. Los perfiles de un solo disparo personalizados pueden incluir bundles de Host o Web explícitamente, mientras que el perfil enviado y la tupla propiedad de la instalación reconocida están libres de Web.
