# Agent Note: Punto de sesión completada en la barra lateral

Status: implemented

[English](2026-08-06-session-completed-done-dot.md) | Español

## Problema

Una sesión a la que el operador delegó trabajo y luego abandonó (cambió a otra conversación) no da ninguna señal cuando termina. Su indicador de ejecución se detiene, pero la fila se ve entonces idéntica a cualquier sesión en reposo, así que el operador debe hacer polling de la lista o descubrir tarde el trabajo terminado. El punto ámbar de interacción pendiente cubre las sesiones que necesitan entrada, no las sesiones cuyo trabajo simplemente está hecho.

## Decisión

`SessionManager` es dueño de un conjunto de recordatorios de finalización del lado del cliente, hermano del bit de interacción pendiente: una arista running→idle de una sesión que no es la seleccionada arma su recordatorio; `select()`/`selectSubagent()` lo consumen; iniciar una nueva ejecución lo desarma y su finalización lo re-arma; la eliminación lo poda. El bit viaja en `SessionListEntry` → `SessionSummary` (opcional, ausente = sin recordatorio) al workspace browser, cuyas filas de sesión y búsqueda renderizan el estado `done` existente de `StateDot` — running conserva el spinner en curso, una sesión en reposo sin recordatorio no muestra nada — y cuya tarjeta hover etiqueta el recordatorio 已完成 / Completed.

El recordatorio está en memoria y es por navegador. Sobrevive a las generaciones de conexión — un blip de transporte no invalida «todavía no has mirado» — pero no a una recarga de página.

## Consecuencias

Los estados de fila de la barra lateral pasan a ser tres señales disjuntas: verde = terminada y sin ver, ámbar = esperando la entrada del operador, azul = en ejecución. Ningún formato de wire, en disco o de configuración cambia: `SessionSummary.completed` es opcional, así que los consumidores existentes y los fixtures de test siguen siendo válidos, y solo el workspace browser lo lee. La arista de finalización se detecta con avidez en cada mutación y pull de la lista (un pase solo en tiempo de construcción de snapshot colapsaría dos frames de estado consecutivos en una observación y perdería la finalización).

## Alternativas consideradas

- **Estado de UI local al componente.** Rechazado porque la barra lateral se desmonta al colapsar y múltiples superficies (árbol agrupado, lista plana, búsqueda) necesitan el mismo bit; el manager ya es dueño de las transiciones de ejecución y de la selección, así que un conjunto propiedad del manager es la única fuente que todas las superficies pueden proyectar.
- **Armado dirigido por eventos solo desde los frames de estado.** Rechazado porque un pull de la lista puede llevar también una transición running→idle (una sesión terminó mientras el refresh estaba en vuelo); el recordatorio se reconcilia contra cada mutación y pull.
- **Persistir el recordatorio.** Rechazado porque el recordatorio significa «todavía no has mirado esta sesión» en este navegador; la recarga restaura la selección y el usuario vuelve a estar mirando la lista, así que un bit durable solo se quedaría obsoleto.
