# Agent Note: Los hijos bifurcados siguen siendo de un solo disparo

Status: implemented

[English](2026-08-10-fork-children-stay-one-shot.md) | Español

## Problema

La única diferencia de Fork respecto a spawn es que la Session hija se siembra con el prefijo de turnos completados del padre ([subagent-fork-in-process](../../../../packages/subagent/subagent-fork-in-process/README.md)). Esa siembra cuesta tokens reales —el historial heredado se reenvía en cada petición hija— y su única recompensa concreta es la reutilización de prefijo del lado del provider: bajo el mismo provider y modelo, una petición hija cuyos bytes iniciales son idénticos a los del padre no re-prefilla ningún tramo compartido. Cualquier cosa que un ámbito hijo añada *por delante* del historial heredado gasta esa recompensa, porque la reutilización se detiene en el primer byte distinto.

El canal de retorno `report` con ámbito hijo es ahora la mayor adición de este tipo, y desde [la obligación de report](../feature/2026-08-06-continuable-child-report-obligation.es.md) son dos deltas en lugar de uno: el schema de la herramienta `report` y la sección `tool:report` del prompt del sistema. Ambos viven en la cabecera de la petición —el bloque del sistema y el bloque de herramientas preceden a cada mensaje— de modo que un hijo bifurcado continuable invalida la reutilización antes del primer turno heredado y re-prefilla todo el transcript que se bifurcó para reutilizar. Esa composición paga el coste de duplicación de fork y no recoge ninguno de sus beneficios, mientras el padre sigue manteniendo un prefijo reutilizable que el hijo podría haber compartido.

## Decisión

Toda composición enviada enlaza la herramienta de delegación fork a `backgroundMode: one-shot`: [el bundle base](../../../../packages/bundle/base/cordis.patch.yml), [el ejemplo ACP](../../../../examples/acp-agent/cordis.yml) y [el ejemplo headless](../../../../examples/headless-agent/cordis.yml). El bundle base deja `run_in_background` disponible, porque monta un servicio de tareas; los dos ejemplos fijan `enableRunInBackground: false`, porque no montan ninguno y un arranque de fondo de un solo disparo fallaría en el momento de la llamada por un servicio `tasks` ausente.

Los hijos de un solo disparo —de primer plano y de fondo por igual— se crean a través de `SubagentRuntime.start()`, que nunca entra en el registro continuable de configuración de activación, así que no se instala ni `report` ni su sección de prompt. El prompt del sistema y los schemas de herramienta de un hijo bifurcado de un solo disparo igualan por tanto a los de su padre, aparte de los deltas de `persona` y `toolFilter` en los que un despliegue opta por herramienta de delegación.

`spawn` conserva `backgroundMode: continuable`. Los hijos continuables y la obligación de report se envían sin cambios para el provider cuyo hijo arranca sin prefijo heredado que proteger, de modo que esta decisión no cuesta nada al canal de report.

### La restricción es de composición, no de código

`ForkInProcessProvider.prepareContinuable` sigue implementado y `ctx.subagents.startContinuable()` sigue aceptando `fork`; solo cambiaron las filas `cordis.yml` enviadas. `tool-subagent` conoce tanto `inheritsParentContext` del provider como su propio `backgroundMode` en el montaje, así que un rechazo en tiempo de carga del par era posible y deliberadamente no se añade: el par no es erróneo en general. Solo lo es mientras un delta de ámbito hijo precede al historial heredado, y el paquete que crea ese delta —[`dsh-tool-subagent-report`](../../../../packages/subagent/tool-subagent-report/README.md)— es instalable por separado e invisible para `tool-subagent` por su propio diseño. Un despliegue que omita el paquete de report puede ejecutar hijos bifurcados continuables con el prefijo intacto. Codificar la consecuencia de una plantilla como invariante de la herramienta de delegación haría que la herramienta afirmara algo que no puede observar.

La condición de reintroducción se registra como marcador `TODO(fork-continuable-prefix-reuse)` en el propio `prepareContinuable`, el único método que las composiciones enviadas no llaman, y se sigue como issue #2124: el fork continuable se reabre cuando el prompt del sistema y los schemas de herramienta de un hijo puedan igualar byte a byte a los de su padre.

## Alternativas consideradas

**Rechazar `inheritsParentContext` + `continuable` en el montaje.** Un fallo ruidoso en tiempo de carga impediría la reintroducción silenciosa, que es lo que el cambio de configuración no puede hacer. Rechazado porque la herramienta de delegación no puede ver el paquete de report y la combinación es legítima sin él; el invariante sería falso para un despliegue que nunca instala un delta de ámbito hijo, y `tool-subagent` estaría afirmando un hecho que pertenece a la plantilla.

**Dejar de montar del todo el provider de fork.** Esta era la forma más amplia de la restricción. Rechazado porque el fork en primer plano *es* el caso de reutilización de prefijo y el canal de report no lo toca, así que una prohibición total renuncia a la capacidad sin comprar nada que el enlace de un solo disparo no compre ya — y dejaría a ninguna composición enviada ejercitando la siembra de sesión.

**Enviar hijos bifurcados continuables y aceptar la pérdida.** Rechazado porque la pérdida es total en lugar de marginal: la reutilización se rompe antes del historial heredado, así que el hijo paga el prefill completo por un transcript que duplicó con el único propósito de no pagarlo. Un despliegue que quiera un hijo longevo sin contexto heredado ya tiene `spawn`.

**Hacer `report` visible para todo Agent.** Un registro global restauraría los prefijos byte-idénticos dando al padre y al hijo el mismo schema y la misma sección. Rechazado porque las raíces, los hijos de un solo disparo, los hijos remotos y los llamantes sin agent anunciarían una herramienta sin destinatario derivable, y el rechazo en tiempo de ejecución haría que la visibilidad del schema discrepara de la autoridad — la decisión local de ámbito que la [Agent Note de la herramienta de report](../feature/2026-07-30-continuable-subagent-report-tool.es.md) ya zanjó.

**Instalar los deltas de ámbito hijo después del historial heredado.** Rechazado como irrepresentable: el prompt del sistema y los schemas de herramienta son estructuras de cabecera de petición en el formato de cable de todo provider, así que ningún orden dentro de ellos puede colocar una adición solo-hijo detrás de la lista de mensajes.

## Consecuencias

- Ninguna composición enviada crea un hijo bifurcado continuable; `subagent_fork` devuelve un resultado al turno de su llamante, y `send_message` se dirige solo a los hijos spawnados.
- El prefijo de petición de un hijo bifurcado permanece byte-idéntico al de su padre salvo que el despliegue configure `persona` o `toolFilter` en la herramienta de delegación fork, de modo que el coste de tokens de la siembra vuelve a comprar la reutilización del lado del provider.
- La vía continuable del provider de fork no tiene llamante de producción ni cobertura de composición ensamblada. Conserva sus tests a nivel de paquete, y el seam sigue aceptándola, así que un bundle o un overlay `--patch` puede reintroducirla sin cambio de código ni aviso.
- El schema visible para el modelo de `subagent_fork` cambia: la redacción continuable de fondo se sustituye por la redacción de tarea de un solo disparo en el bundle base, y desaparece por completo de los dos ejemplos. Los sidecars afectados de schema de herramienta de la instantánea sin clave se re-graban en el mismo cambio.
- El alcance de la obligación de report se estrecha a los hijos spawnados en los despliegues enviados. Su planificación por defecto `next-step`, su modelo de autoridad y su cobertura siguen siendo independientes de la composición de fork.

### Riesgos aceptados

La restricción vive en tres archivos de configuración y un comentario de código, no en un control. Una futura fila de bundle o parche de perfil puede fijar `backgroundMode: continuable` en una herramienta fork y reintroducir silenciosamente la pérdida de prefijo; nada falla de forma ruidosa. Ese es el coste aceptado de no codificar la consecuencia de una plantilla en `tool-subagent`.
