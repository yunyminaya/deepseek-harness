# Agent Note: La tira de plan todo se limpia en el siguiente turno

Status: implemented

[English](2026-07-28-todo-plan-clears-on-next-turn.md) | Español

## Problema

`todo_write` almacena instantáneas de la lista completa en el log de la sesión, y los hosts interactivos renderizan la última lista como una tira de plan (TodoPanel web a través de la proyección `todos`, panel Plan de la TUI). Cuando un turno terminaba, esa tira seguía en pantalla durante el siguiente turno del usuario: una lista completada o abandonada de la tarea anterior. Los lectores tratan la tira como «lo que está haciendo este turno», así que una lista obsoleta a través del límite de turno es la vida útil de producto equivocada. Las notas de [visualización web del todo](2026-07-23-web-todo-display.es.md) y de la [herramienta `todo_write`](2026-06-29-todo-write-tool.es.md) siguen siendo dueñas del event sourcing y de las dos superficies de render; describían el plan vigente como duradero durante toda la sesión hasta la siguiente escritura.

## Decisión

El plan vigente es el último `todo/write` que no va seguido de un `turn/start` posterior. `turn/end` mantiene la lista visible para que la lista completada permanezca mientras el usuario lee la respuesta; el siguiente `turn/start` la limpia hasta que el modelo vuelva a escribir.

### Proyección del host (web)

La proyección unitaria `todos` de `dsh-tool-todo` pliega la regla: `apply` toma la lista completa de cada `todo/write` y devuelve `null` en cada `turn/start` (`stateVersion` 2). Los carriers (`dsh-host-apiproxy`) sirven ese valor en el bloque `projections` de la cola del historial y empujan frames `session/projection`; el dock web lo lee a través de `useProjection('todos')`. El fixture sin clave refleja el mismo pliegue para las instantáneas ensambladas.

### Ruta en vivo de la TUI

El `renderEvent` switch de la antigua TUI limpiaba su panel de plan local en `turn/start` y lo reemplazaba en `todo/write`, con su ruta de reconstrucción reiniciando el panel antes del replay para que una reanudación en frío convergiera en la misma regla; ese paquete se ha eliminado desde entonces ([eliminación del paquete TUI](../simplification/2026-08-04-remove-tui-package.es.md)).

## Alternativas consideradas

- **Limpiar en `turn/end`** — oculta la lista mientras el usuario sigue leyendo la respuesta recién terminada; el trabajo de la tira en ese momento es el plan completado, no un dock vacío.
- **Limpiar solo cuando todos los elementos estén `completed`** — deja planes abandonados o parciales entre turnos; la tira seguiría mostrando el trabajo de otra tarea.
- **Añadir un `todo/write` vacío al inicio del turno** — muta el log por una regla de vida útil de la UI e inventa una escritura que el modelo nunca hizo.

## Consecuencias

La proyección del host y el panel de la TUI comparten una única regla de vida útil; reabrir una sesión restaura un plan solo cuando no ha comenzado ningún turno posterior. Supersesión parcial de la redacción de plan vigente para toda la sesión en [visualización web del todo](2026-07-23-web-todo-display.es.md) y en la [herramienta `todo_write`](2026-06-29-todo-write-tool.es.md): el event sourcing, el reemplazo último-escritura-gana y las dos superficies de render permanecen allí; esta nota es dueña de la limpieza en el límite de turno. Cobertura: specs de la proyección tool-todo para limpiar en turn/start + conservar en turn/end, limpieza de frames push del fixture para la instantánea web ensamblada, más la instantánea de TUI que inicia el siguiente turno y fija la tira desaparecida.
