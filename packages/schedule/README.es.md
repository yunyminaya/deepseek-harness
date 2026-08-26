# schedule/ — Recordatorios locales a la sesión

[English](README.md) | Español

La familia Schedule es dueña de los recordatorios cuyo estado durable vive en el log de la Session original. Un dueño local al proceso espera solo mientras esa Session tiene un root Agent en vivo; las Sessions frías reanudan el trabajo vencido cuando vuelven a estar en vivo y nunca implican un canal de notificación externo.

| Paquete | Rol | clave ctx |
|---|---|---|
| `schedule/` | Eventos y fold de Schedule versionados, herramientas create/list/delete orientadas al modelo, y un dueño de temporizador de root Agent en vivo | — |

El paquete no expone deliberadamente ningún servicio público de Schedule ni base de datos mutable. Las herramientas y el runtime hacen append al flujo de la Session; el trabajo debido entra en la misma conversación a través de la cola ordinaria de seguimiento del Agent.

Consulta [Schedule local a la sesión](../../docs/subsystems/schedule.es.md) para los contratos de registro durable, transición, vista y entrega.
