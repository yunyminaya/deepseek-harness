# Agent Note: `list_agents` usa `ready` para los hijos reanudables

[English](2026-08-06-list-agents-residency-vocabulary.md) | Español

Estado: implementado

## Problema

`list_agents` proyectaba la residencia de proceso de un hijo continuable como `running | idle | complete`. `complete` se lee como una unidad de trabajo terminal con un resultado en alguna parte, pero el hecho subyacente solo dice que no hay ninguna Activation residente: la conversación está intacta, `send_message` puede continuarla y no se afirma nada sobre el resultado del hijo. Un modelo que lee `complete` busca razonablemente un resultado que recoger o envía trabajo de sustitución a una conversación que cree terminada.

La palabra es especialmente engañosa junto a la [entrega de la resolución propiedad del gestor](../feature/2026-08-06-manager-owned-subagent-settlement-delivery.es.md). La finalización llega al padre como un aviso; el listado existe para recordar conversaciones persistentes, no para sondear ese aviso.

## Decisión

La proyección orientada al modelo informa de `running | idle | ready`:

- **`running`** significa que el agent (agente) residente tiene un driver activo.
- **`idle`** significa que el agent está residente entre turnos y puede estar esperando a agents que inició.
- **`ready`** significa que solo queda la conversación persistente. `send_message` inicia el siguiente turno en la misma conversación; el estado es reanudable, no terminal, y no significa que haya un resultado esperando a ser recogido.

La descripción de la herramienta establece esas distinciones y aleja al modelo del sondeo: dice que el padre es informado cuando un hijo termina y que el listado sirve para recordar qué hijos inició. `send_message` sigue siendo la comprobación de entrega autoritativa porque cualquiera de las dos instantáneas puede competir con otro proceso o con un mensaje posterior.

La capa de servicio no cambia. `SubagentListEntry.activity` conserva `'running' | 'inactive'`, que describe con precisión la residencia en el corpus para consumidores como una interfaz. El adaptador orientado al modelo asigna `inactive` a `ready` porque esa palabra comunica la acción disponible para el modelo sin inventar un resultado.

## Alternativas consideradas

**Conservar `complete` y matizarlo en la descripción.** Una descripción que diga que `complete` no significa complete lucha contra el estado renderizado en cada lectura. La línea que escanea el modelo debe portar la distinción correcta por sí misma.

**Usar `active | dormant`.** Elimina la distinción útil entre un agent residente que está entre turnos y una conversación solo-almacenamiento, y hace que el estado solo-almacenamiento suene no disponible. `ready` expresa el hecho útil: la misma conversación acepta otro turno.

**Eliminar el estado por completo.** La residencia sigue siendo útil cuando un padre decide si enviar más trabajo. Eliminarla cambia un estado engañoso por ninguna señal.

**Renombrar los valores de actividad del servicio.** `running | inactive` es correcto en la capa de servicio y tiene consumidores que no son el modelo. Renombrarlo removería un contrato general para arreglar la presentación de un adaptador; la [nota del catálogo persistente](../feature/2026-07-22-durable-subagent-catalog-and-list-agents.es.md) sigue siendo propietaria de ese vocabulario de servicio.

## Consecuencias

- La línea renderizada usa `<id> [running] — <label>`, `<id> [idle] — <label>` o `<id> [ready] — <label>`.
- El enum `status` del schema de salida cambia con el contrato renderizado. El catálogo de herramientas generado incorpora la nueva descripción; renderiza solo los `parameters` de cada herramienta y nunca llevó el schema de salida.
- La cobertura unitaria fija las tres asignaciones y las cláusulas de descripción que dirigen al modelo al aviso de resolución en lugar de sondear esta herramienta.
- El escenario ACP ensamblado `subagent-list-agents` renderiza `ready` para un hijo resuelto y reanudable.
