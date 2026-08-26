# Agent Note: Los informes de subagente preceden a sus avisos de cierre

Status: implemented

[English](2026-08-17-subagent-report-settlement-ordering.md) | Español

## Problema

Un hijo continuable puede informar explícitamente de contenido seleccionado y, más tarde, producir un aviso de cierre incondicional redactado por el gestor. La entrega del informe usaba `Agent.followup()` y entraba en la cola `next-turn` del padre, mientras que la entrega del cierre a un padre en ejecución usaba `Agent.steer()` y entraba en `next-step`. El primer paso de un turno reclama el lote completo de `next-step` antes que un mensaje de `next-turn`, así que el aviso de cierre posterior podía llegar al modelo antes que el informe anterior. El escenario ensamblado de informe requería `reportDelivery: quiet` para evitar esa intercalación no determinista. [El issue #2600](https://github.com/deepseek-harness/deepseek-harness/issues/2600) registra el defecto.

La herramienta de informe indica a un hijo que informe siempre que un hallazgo cambie lo que su padre debería hacer a continuación. Aplazar ese mensaje a un turno posterior contradecía el significado de programación de la herramienta y separaba mensajes ordenados causalmente en colas con distinta prioridad de reclamación.

## Decisión

`SubagentReportDelivery` es `'quiet' | 'next-step'`, y `next-step` es el valor por defecto. La entrega next-step llama a `parent.steer()`, de modo que un padre en ejecución lee el informe en su límite de paso seguro más próximo y un padre inactivo inicia un turno. La entrega quiet sigue llamando a `parent.inject()` y entra en la misma cola sin despertar a un padre inactivo.

El gestor de continuación conserva `sendWaking()` y `admitWaking()` alrededor de los informes next-step entregados a padres continuables residentes. Su propósito es la contabilidad de admisión de envíos de activación, con independencia de si el mensaje apunta a un paso o a un turno: la Activation receptora sigue viva entre la inserción síncrona en la bandeja de entrada y la microtarea que observa la activación.

### Orden entre estados del padre

Un padre en ejecución recibe un informe aceptado y el aviso de cierre posterior del hijo en el mismo FIFO `next-step`. Si el padre queda inactivo antes de que llegue el cierre, ya ha reclamado el informe; el cierre puede entonces abrir un turno posterior sin invertir el orden observado.

Durante el mantenimiento del padre, el informe ocupa `next-step` y fija una activación, mientras que el cierre puede ocupar `next-turn` porque el mantenimiento informa del estado inactivo. La reclamación inicial sigue tomando la entrada de next-step antes que el turno encolado. La entrada de activación enviada tras la cancelación es redirigida por `Agent.send()` a `next-turn`, de modo que informe y cierre siguen la convergencia de cancelación del agent central en lugar de eludirla.

### Verificación

El paquete de informe mantiene un padre dentro de una petición de modelo activa, envía un informe de hijo, cierra ese hijo y verifica que el lote pendiente del padre esté ordenado `subagent-report` y luego `subagent-settled`, sin ningún turno posterior encolado. Una cobertura aparte fija los informes repetidos como un único lote FIFO de next-step, el despertar de un padre inactivo y la contabilidad de admisión de activación para un padre continuable.

El escenario ensamblado de informe ACP usa el valor por defecto publicado. Su valla de programación mantiene al hijo detrás del turno de delegación del padre y retiene al padre en mantenimiento hasta que el cierre sigue al informe. El informe fija la activación mientras el aviso de cierre encola un turno; cuando el mantenimiento termina, el padre reclama la entrada de next-step antes que la de next-turn y observa ambos avisos en orden causal sin una superposición de entrega quiet.

## Alternativas consideradas

**Mantener el nombre `wakeup` pero cambiar su implementación a `steer()`.** La descripción pública existente definía `wakeup` como un turno posterior del padre. Reutilizar el valor para un destino de bandeja distinto dejaría a la configuración incapaz de expresar el comportamiento que selecciona. La configuración pre-release nombra en su lugar `next-step` directamente.

**Exponer `quiet | next-step | next-turn`.** Un informe next-turn todavía permite que un aviso de cierre next-step posterior lo adelante. Preservar informe-antes-de-cierre exigiría una barrera de orden entre colas, y ningún despliegue actual necesita aislamiento next-turn con suficiente fuerza como para ser dueño de ese mecanismo.

**Mover los avisos de cierre a `next-turn`.** El agrupamiento de cierres usa deliberadamente la cola next-step para que varios hijos que terminan juntos cuesten un paso del padre en lugar de un turno cada uno. Mover el cierre aumentaría la latencia y el trabajo del modelo para conservar un modo de programación de informes sin ningún consumidor actual.

## Consecuencias

- Un informe puede extender un turno abierto del padre. Nunca interrumpe la petición de modelo activa ni la ejecución de herramientas; el agent loop solo lo admite en un límite de paso.
- Los informes aceptados juntos comparten un único lote next-step, preservando el orden FIFO y reduciendo la amplificación de turnos del antiguo comportamiento de un turno por informe.
- El valor de configuración `wakeup` se rechaza en lugar de conservarse como alias. Este repositorio no tiene ninguna promesa de compatibilidad pre-release externa para la configuración de Cordis.
- `quiet` sigue siendo la vía de escape del despliegue para informes que no deben despertar a un padre aparcado, con el riesgo existente de que ningún modelo los lea hasta que llegue otra entrada de activación.
