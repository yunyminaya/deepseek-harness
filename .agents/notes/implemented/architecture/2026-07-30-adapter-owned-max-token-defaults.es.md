# Agent Note: Valores predeterminados de max-tokens propiedad del adaptador

Status: implemented

[English](2026-07-30-adapter-owned-max-token-defaults.md) | [中文](2026-07-30-adapter-owned-max-token-defaults.zh.md) | Español

## Problema

Un adaptador de LLM podía serializar un `GenerateOptions.maxTokens` explícito, pero su configuración de Cordis no podía establecer un valor predeterminado de conversación reconstruible. Aplicar un respaldo solo dentro de la serialización del provider haría que la petición wire difiriera de la cabecera durable `request/header`; poner el valor predeterminado de cada provider en Agent Loop trasladaría en cambio la política de despliegue y de modelo al driver neutral respecto al provider.

## Decisión

`LlmResolvedModelInfo.defaultMaxTokens` lleva un tope de salida por petición opcional, configurado por el adaptador, para una ruta exacta de provider/modelo. `LlmRuntime` lo valida como entero seguro positivo y lo materializa en `LlmCallConfig.maxTokens` solo cuando el llamante omitió un valor. Una llamada preparada identifica los campos materializados `maxTokens` y `reasoningEffort` como valores predeterminados del adaptador; las opciones explícitas de la petición o del Agent permanecen sin marcar y por tanto ganan sin recorte.

El agent loop sigue preparando las llamadas antes de registrar `request/header`, por lo que la config efectiva y los marcadores de los campos suministrados por los valores predeterminados del adaptador se convierten en hechos durables de la petición antes del despacho. Antes del siguiente waterfall de `agent/request`, el loop elimina de la propuesta los campos marcados; la resolución exacta de modelo materializa entonces de nuevo los valores predeterminados de la ruta actual. Un cambio de provider/modelo no puede por tanto confundir el valor predeterminado de un adaptador anterior con una anulación explícita, mientras que los valores explícitos de la conversación persisten. Las llamadas directas a `LlmRuntime.stream()` resuelven el mismo valor predeterminado en la frontera final del adaptador. El campo es un valor predeterminado de petición, no un límite duro de salida del modelo; los adaptadores que conservan los valores predeterminados propiedad del provider lo omiten.

El adaptador nativo de DeepSeek expone `maxTokens` en la config de Cordis con un valor predeterminado de 256,000 tokens y asigna el valor efectivo a `max_tokens`. Su capacidad de contexto predeterminada es de 1,000,000 tokens: ambas entradas V4 integradas publican esa capacidad exacta, mientras que las entradas configuradas sin capacidad y los ids de paso directo no listados heredan el mismo respaldo general del adaptador.

## Alternativas consideradas

**Aplicar el valor predeterminado solo en la serialización de DeepSeek.** Rechazada porque el wire del provider contendría un valor visible para el modelo ausente de la cabecera durable de la petición.

**Establecer `AgentOptions.maxTokens` en cada aplicación distribuida.** Rechazada porque las aplicaciones duplicarían la política de despliegue del adaptador, las llamadas directas al LLM se comportarían de forma distinta y seleccionar otro provider conservaría un tope específico de DeepSeek.

**Representar 256,000 como un máximo duro por modelo.** Rechazada porque el valor configurado es el presupuesto de petición deseado, no una prueba de que cada endpoint configurado rechace salidas mayores. Los llamantes explícitos siguen siendo autoritativos.

**Dejar el valor predeterminado del provider al control.** Rechazada para el despliegue nativo de DeepSeek porque el producto exige un presupuesto de conversación estable de 256,000 tokens entre endpoints compatibles.

## Consecuencias

Las conversaciones de DeepSeek envían `max_tokens: 256000` por defecto, y la cabecera de petición de la sesión registra tanto el valor como que el adaptador lo suministró. Los despliegues pueden cambiar el valor predeterminado del adaptador mediante `llm-deepseek.config.maxTokens`; los valores por agent y por petición lo anulan. Cambiar de ruta rematerializa el valor predeterminado del nuevo adaptador exacto en lugar de arrastrar el valor derivado de DeepSeek. Los demás adaptadores conservan su comportamiento existente hasta que publican intencionadamente `defaultMaxTokens`.

El presupuesto de salida de 256,000 tokens reserva una gran parte del contexto de un millón de tokens en los endpoints que preasignan la salida solicitada. Los despliegues cuyo gateway o modelo admita un presupuesto menor deben bajar `maxTokens`; la configuración explícita es preferible a un respaldo no documentado del provider.
