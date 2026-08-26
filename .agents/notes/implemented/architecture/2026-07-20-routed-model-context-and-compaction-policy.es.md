# Agent Note: Política de contexto de modelo enrutado y compactación

Status: implemented

[English](2026-07-20-routed-model-context-and-compaction-policy.md) | Español

## Problema

La compactación no puede aplicar con seguridad una única ventana de contexto global cuando un proceso enruta peticiones a modelos con capacidades distintas. El mismo id de modelo también puede existir bajo varios providers, y un adaptador puede aceptar ids dinámicos ausentes de su catálogo informativo. Una capacidad equivocada compacta demasiado tarde y dispara desbordamientos evitables, o compacta demasiado pronto y descarta contexto útil.

Ninguno de los dos propietarios de configuración obvios es suficiente. Compact-basic es opcional y no sabe qué modelos acepta un adaptador. Los adaptadores LLM (modelo de lenguaje de gran tamaño) son dueños del enrutamiento de modelos pero no deben depender de un plugin de compactación opcional ni absorber la política de umbral, retención, resumidor y reintentos específica del consumidor. El diseño necesita un hecho de capacidad autoritativo y una política de compactación opcional por destino sin crear un segundo registro de modelos.

## Decisión

### Los adaptadores son dueños de la capacidad de ruta exacta

`LlmAdapter.resolveModel(provider, model, signal?)` devuelve metadatos agregados para una ruta exacta, con `LlmModelContext` opcional bajo su campo `context`. `LlmRuntime.resolveModelInfo()` selecciona al propietario de ruta registrado, valida un `contextWindow` entero positivo y devuelve metadatos desacoplados. La consulta es independiente de `listModels()`: un modelo dinámico no listado puede tener metadatos de capacidad, y un `context` ausente solo significa que el adaptador no puede describir la capacidad.

El adaptador DeepSeek escrito a mano acepta `contextWindow` opcional en cada modelo configurado además de un `defaultContextWindow` de todo el adaptador. La capacidad exacta del modelo gana; una entrada sin capacidad y un id pass-through no listado heredan el valor por defecto del adaptador, u omiten `context` cuando está ausente. Las dos entradas de modelo integradas publican cada una una capacidad exacta de 256,000 tokens. El adaptador pi-ai resuelve la capacidad a partir del mismo descriptor de catálogo que resuelve de forma autoritativa el modelo de la petición.

### La medición de tokens sigue siendo agnóstica al modelo

`dsh-token-meter` no tiene configuración ni perfiles de modelo. Es dueño de un único plegado de replay fijo y devuelve presión de tokens estimada absoluta más precios posicionales de superficie. Eliminar la capacidad global mantiene la medición reutilizable cuando compact-basic está ausente e impide que la contabilidad de replay se convierta en otro registro de modelos.

### Compact-basic resuelve una especificación de destino

Compact-basic es dueño de la política del consumidor. Los campos de nivel superior definen los valores por defecto; `modelPolicies` contiene anulaciones parciales indexadas por el par exacto `{ provider, model }`. Los destinos duplicados y los campos desconocidos o inválidos fallan la carga del plugin. `thresholdRatio` por defecto es `0.8`, y la retención por defecto es `retainRatio: 0.16`; los llamadores pueden usar en su lugar un `retainTokens` absoluto, pero las dos formas de retención son mutuamente excluyentes. Tras la herencia, una retención por ratio que no esté por debajo de su ratio de umbral también falla la carga del plugin porque ninguna capacidad de modelo puede hacer válida esa política.

Para la presión proactiva, compact-basic lee la última ruta de petición durable, resuelve la capacidad del adaptador y la política de destino exacto, y escala los ratios a un `ResolvedCompactSpec`. Realiza esta resolución en cada comprobación, de modo que un cambio de provider o de modelo en una sesión cambia la capacidad y la política de inmediato. Un presupuesto de retención absoluto que no esté por debajo del umbral escalado falla cuando la capacidad del destino hace posible esa comparación por primera vez.

La misma anulación de destino exacto puede seleccionar el provider/modelo de resumen, el tope de salida del resumen, los reintentos de convergencia y el tope de reintentos de desbordamiento. Son asuntos de compactación y nunca entran en un provider LLM.

### Los fallos de presión específicos de destino conservan la composición opcional

Un adaptador sin metadatos de capacidad sigue siendo una ruta LLM válida. La presión proactiva manual falla con un error de configuración específico de destino; el listener automático advierte una vez por ruta exacta y continúa con la historia completa. La misma supresión por ruta se aplica cuando la capacidad resuelta expone un presupuesto de retención absoluto inválido, mientras que los fallos operativos no relacionados permanecen visibles de forma independiente. El desbordamiento canónico confirmado por el provider no necesita metadatos de capacidad: elude el umbral proactivo y el presupuesto de retención normal, intenta una reducción equilibrada máxima única y conserva el error original del provider a menos que el reemplazo demuestre progreso.

## Pruebas

Los tests de servicio cubren los metadatos de contexto desacoplados, la salida de adaptador inválida, la independencia del catálogo y la ausencia del valor por defecto. Los tests de adaptador cubren la resolución DeepSeek exacta/por defecto/no listada, las capacidades inválidas y la resolución pi-ai del descriptor exacto. Los tests de compactación cubren el escalado de ratios, las anulaciones exactas de provider/modelo, el rechazo en carga de ratios fusionados inválidos, la validación de presupuesto absoluto en runtime, los cambios de provider con el mismo id de modelo, la supresión de advertencias específica de destino y la recuperación de desbordamiento independiente de capacidad. Los fixtures del loader rechazan el ajuste de capacidad de token-meter eliminado, y los ejemplos configuran capacidad en los adaptadores.

## Alternativas consideradas

- **Poner la capacidad y todas las políticas en compact-basic** — rechazado porque compact-basic duplicaría el conocimiento de modelos de los adaptadores, los modelos dinámicos no listados exigirían registro paralelo, y la capacidad desaparecería cuando la compactación no está instalada.
- **Poner la política de compactación en cada adaptador LLM** — rechazado porque los adaptadores deben permanecer independientes de los consumidores opcionales, mientras que la política de resumen y reintentos no es un hecho del provider.
- **Hacer autoritativo `listModels()`** — rechazado porque el descubrimiento es informativo y algunos adaptadores aceptan deliberadamente ids dinámicos. Los metadatos de corrección no deben convertir la pertenencia al selector en una lista blanca de enrutamiento.
- **Añadir plegados por modelo a token-meter** — rechazado porque el algoritmo de replay es compartido; solo cambian la capacidad y la política del consumidor. Varios plegados duplicarían estado sin mejorar la estimación.
- **Crear un registro independiente de contexto de modelo** — rechazado porque el adaptador ya es dueño de la resolución de ruta autoritativa. Un segundo registro introduciría problemas de orden de ciclo de vida, claves duplicadas y deriva sin un backend independiente.

## Consecuencias

- La capacidad tiene un único propietario autoritativo en el contrato del provider, mientras que la política de compactación permanece en el plugin consumidor opcional.
- La misma instancia de compact-basic maneja con seguridad ventanas distintas, cambios de provider e ids de modelo idénticos bajo providers distintos sin consultar los metadatos de descubrimiento.
- Las composiciones solo-LLM y solo-medidor siguen siendo válidas; cargar compact-basic no añade ninguna dependencia inversa desde los adaptadores.
- Los despliegues DeepSeek pueden fijar capacidades exactas por modelo, o usar `defaultContextWindow` para las entradas sin capacidad y los ids pass-through no listados.
- Los valores por defecto por ratio escalan de forma natural entre modelos, mientras que la retención absoluta de destino exacto sigue disponible para comportamiento específico de despliegue.

Esta nota supera las partes de capacidad global y sin política de modelo de la [Agent Note del servicio de medición de tokens de replay](2026-07-15-replay-token-meter-service.es.md). Su decisión de medición de un solo plegado permanece sin cambios.
