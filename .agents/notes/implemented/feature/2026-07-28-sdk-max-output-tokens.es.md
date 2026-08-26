# Agent Note: Tope de tokens de salida del SDK

Status: implemented

[English](2026-07-28-sdk-max-output-tokens.md) | Español

## Problema

Los SDK de Python y TypeScript podían seleccionar un provider y un modelo, pero no podían acotar la salida del modelo de conversación. El runtime omitía por tanto `GenerateOptions.maxTokens`, dejando que los valores por defecto del provider mandaran incluso cuando un host de evaluación exigía un presupuesto de salida fijo. `compaction-basic.maxTokens` no podía cubrir ese papel porque limita solo las llamadas de resumen de compactación.

## Decisión

Los SDK de alto nivel exponen un único tope opcional de todo el proceso: Python lo llama `max_tokens`, TypeScript lo llama `maxTokens`, y el payload de cable compartido de `initialize` lleva `maxTokens`. El servidor JSON-RPC rechaza los valores que no son enteros seguros positivos y almacena el tope aceptado junto con su ruta de provider/modelo.

Cada Agent raíz creado por el SDK recibe el tope a través de `AgentOptions.maxTokens`. Agent Loop coloca ese valor en el `LlmCallConfig` inicial; la preparación de la llamada final conserva el valor explícito o materializa un valor por defecto del adaptador exacto del modelo, registra el tope efectivo en la cabecera de la solicitud y reconstruye cada solicitud de conversación enviada a partir de esa cabecera duradera. Omitir la opción del SDK permite por tanto que se aplique el valor por defecto del adaptador o de la ruta de provider seleccionada.

Los subagentes en proceso heredan el provider, el modelo y el tope de salida del padre. Un `SubagentStartRequest.agentOptions.maxTokens` explícito, incluido uno configurado por `dsh-tool-subagent`, anula el valor heredado para ese hijo y sus descendientes. Los providers fuera de proceso son dueños de la configuración de su runtime separado; `subagent-dsh-sdk` expone por tanto su propio `maxTokens` opcional y lo reenvía a través del handshake del SDK de ese runtime hijo.

La compactación, la generación de títulos de sesión, la búsqueda web y otras llamadas auxiliares conservan sus límites de salida de propiedad independiente. `maxTokensAsSuccess` sigue siendo solo mapeo de resultado: no fija ni altera el tope.

## Alternativas consideradas

**Fijar solo una variable de entorno del adaptador.** Un fallback privado del serializador sería específico del adaptador DeepSeek, invisible en la cabecera de la solicitud de la sesión, ineficaz para los adaptadores interceptados o alternativos y fácil de confundir con un valor por defecto del provider. Los valores por defecto de propiedad del adaptador pueden exponerse en su lugar como metadatos exactos del modelo y materializarse en la configuración de la solicitud neutral respecto al provider antes de registrarse en el log.

**Añadir `maxTokens` a cada `session/prompt`.** La mutación por turno agrandaría el cable e introduciría transiciones de configuración de solicitud que los llamadores no necesitan para el caso de uso de evaluación actual. Una opción de inicialización del runtime da a cada sesión de un mismo proceso SDK el mismo presupuesto reproducible.

**Reutilizar `compaction-basic.maxTokens`.** El valor de compactación controla la generación de resúmenes, no las solicitudes de conversación ordinarias. Compartirlo acoplaría dos presupuestos de tokens distintos y haría que ajustar uno cambiara silenciosamente el otro.

## Consecuencias

Los llamadores del SDK pueden acotar la salida del modelo sin editar la composición de Cordis, y la creación directa de Agents usa el mismo contrato validado `AgentOptions`. El tope es visible en las cabeceras de solicitud duraderas y llega a los adaptadores de provider como `GenerateOptions.maxTokens`; la serialización de DeepSeek lo mapea a `max_tokens`.

Un runtime del SDK tiene un tope por defecto. Un llamador que necesite topes distintos ejecuta instancias de runtime separadas o anula explícitamente un hijo en proceso a través de sus agent options. Alcanzar el tope sigue produciendo el motivo de parada existente `max-tokens`, cuyo mapeo `ok` o `error` sigue siendo política del despliegue.
