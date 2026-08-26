# Agent Note: El canal de retorno del hijo continuable es una obligación

Status: implemented

[English](2026-08-06-continuable-child-report-obligation.md) | Español

## Problema

Un hijo en segundo plano continuable es dueño de su propia Session, así que nada de lo que escriba allí llega al agent (agente) que lo inició. [La herramienta report](2026-07-30-continuable-subagent-report-tool.es.md) dio a ese hijo un canal de retorno y luego lo presentó como una opción entre varias: el schema decía «llama a esto cero o más veces», nada en el prompt del hijo le pedía llamar a la herramienta, y la programación por defecto aceptada (`quiet`) añadía el report al siguiente request de un padre aparcado sin despertarlo.

Cada una de esas elecciones es defendible por separado. Juntas hicieron que el canal de retorno fuera inutilizable como contrato de delegación. Un hijo que terminaba su trabajo, escribía su respuesta en su propio transcript y se detenía dejaba al padre sin nada; un hijo que sí reportaba llegaba a un padre que ya se había aparcado y no leería el report hasta que algo no relacionado lo despertara. Los reportes externos de padres haciendo busy-poll de `list_agents`, reenviando mensajes a hijos liquidados y abandonando `subagent` por `workflow` se reducen todos a la misma garantía ausente.

## Decisión

El canal de retorno es una instrucción que el hijo recibe, no una capacidad que pueda descubrir. El paquete de report instala dos registros con scope local en cada hijo continuable en proceso, y un solo disposer revoca ambos:

- la herramienta `report`, cuya descripción establece ahora que el hijo la llama una vez antes de terminar con un resultado final autocontenido, y antes, para el progreso que cambia lo que el padre debe hacer a continuación;
- una sección de system-prompt `tool:report` en el orden 117 que lleva la misma obligación en la propia voz del hijo, así que un hijo que nunca lee las descripciones de herramientas con atención la recibe igualmente.

`reportDelivery` tiene por defecto `next-step`. Un report aceptado despierta al driver de un padre aparcado o se une al límite de paso más cercano de un padre en ejecución, en consonancia con la instrucción de reportar hallazgos que cambian la siguiente acción del padre. `quiet` sigue disponible para despliegues que prefieren reports no leídos a la amplificación del trabajo del modelo. La [decisión de ordenamiento report/liquidación](../bug-fix/2026-08-17-subagent-report-settlement-ordering.es.md) es dueña del razonamiento de programación.

### Por qué existen tanto la sección como la descripción

Abordan modos de fallo distintos. La descripción de la herramienta se lee cuando el modelo ya está considerando `report`; la sección de prompt se lee cuando decide si ha terminado. La obligación pertenece a ambos puntos porque el fallo que esto arregla — un hijo que simplemente se detiene — ocurre en el segundo.

La sección se registra en el propio scope del hijo, el mismo mecanismo que la [composición de hijo](../../../../packages/subagent/subagent/src/child-agent.ts) usa ya para una persona que hace sombra, así que el padre y cada hermano no ven ni la herramienta ni la guía. `installReportTool` revierte la sección si falla el registro de la herramienta, y su disposer devuelto intenta ambas revocaciones antes de sacar a la superficie los fallos de limpieza.

### Instrucción, no imposición

Nada rechaza a un hijo que nunca reporta. Ningún camino de runtime inspecciona si se envió un report, y `report` sigue aceptando cero o muchas llamadas por turno. El cambio es redacción orientada al modelo más un valor por defecto de programación; la autoridad del servicio, el acuse de recibo y los contratos de recuperación no cambian.

Ese límite es deliberado: el texto de prompt solo puede llegar a un hijo que sigue ejecutando su propio loop. Un hijo detenido por un error, un techo de tokens, una cancelación o un teardown nunca tiene la oportunidad de cumplir, que es por lo que el runtime mantiene su propia cuenta de la liquidación en lugar de confiar en esta instrucción ([entrega de liquidación propiedad del manager](2026-08-06-manager-owned-subagent-settlement-delivery.es.md)).

### Cobertura de snapshot

El escenario ACP `subagent-report` ensamblado ejercita el valor por defecto incluido: el hijo reporta mientras el padre está en mantenimiento, el aviso de liquidación posterior se encola detrás, y el padre reanudado reclama el report de next-step antes de la liquidación de next-turn. Como el scope del hijo compone un prompt que el class pin no puede describir, el harness de snapshots tiene `pinsChildSystemPrompts`, el contraparte exacto de `pinsChildToolSchemas`: mueve el prompt de un fixture de hijo a `system-prompt.<n>.expected.md`, deja cada otro campo del request-header al class pin, exige el sidecar exactamente cuando se declara y rechaza un sidecar idéntico a ese class pin para que una copia redundante no pueda derivar.

## Alternativas consideradas

**Mantener `quiet` como valor por defecto y confiar solo en el prompt.** Era la posición incluida, y no sustituye nada por sí sola: un report que el padre nunca lee es indistinguible de un report nunca enviado. El [rechazo de despertar siempre de la nota de la herramienta report](2026-07-30-continuable-subagent-report-tool.es.md) asumía que el padre tenía otra razón para mirar su contexto; un coordinador de segundo plano aparcado no la tiene. La amplificación de turnos es el costo real, y ahora es la razón por la que `quiet` sigue existiendo, no la razón por la que es el valor por defecto.

**Dejar que el hijo elija el modo de entrega por llamada.** Sin cambios respecto al rechazo original: el modelo sería dueño de la presión del programador, y el comportamiento variaría por llamada en lugar de por despliegue.

**Poner la obligación solo en la descripción de la herramienta.** Una descripción se lee al elegir entre herramientas. El hijo al que apunta este cambio no está eligiendo una herramienta; cree que ha terminado. La guía de prompt es la superficie que llega a esa decisión.

**Imponer la obligación en la liquidación rechazando a un hijo silencioso.** No hay nada que rechazar: cuando la liquidación es observable, el loop del hijo ha terminado, y fallar su teardown destruiría trabajo en lugar de entregarlo. Entregar los hechos terminales incondicionalmente desde el runtime es la respuesta a ese caso, y pertenece al gestor de continuación, no a este paquete.

## Consecuencias

- Cada hijo continuable en proceso con este paquete cargado lleva una sección de prompt extra y una descripción de `report` más larga en cada request; ningún otro request de ningún Agent cambia.
- El despliegue por defecto despierta al padre una vez por report aceptado. Un árbol anidado que reporta con frecuencia consume requests de padre adicionales, mientras que los reports que esperan juntos comparten un paso; `quiet` es el escape documentado.
- `installReportTool` requiere `ctx.systemPrompt` en el scope del hijo, así que el paquete declara `systemPrompt` en `inject` y falla en la carga en lugar de en la siguiente materialización de hijo.
- La cobertura unitaria fija el nuevo valor por defecto, dos frases de instrucción que soportan carga, el scope solo-hijo de la sección frente tanto al padre como a un hermano, y la reversión o revocación de ambos registros.
- Tres escenarios ACP ensamblados con hijos continuables fijan el texto completo de la instrucción a través del nuevo sidecar; un cambio futuro en cualquier sección de scope de hijo falla esos escenarios en lugar de pasar en silencio.

### Riesgos aceptados

La entrega next-step por defecto amplifica el trabajo del modelo en árboles profundos. El despliegue es dueño de eso mediante `reportDelivery`; los reports que esperan juntos comparten un paso, y un report aceptado causa como máximo un despertar.

Un hijo puede seguir terminando sin reportar, y este cambio no puede detectarlo. Solo la propia [cuenta de liquidación](2026-08-06-manager-owned-subagent-settlement-delivery.es.md) del runtime cierra ese caso.
