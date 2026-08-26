# session-query/ — familia de capacidades de recuperación de sesiones

[English](README.md) | Español

Esta familia proporciona recuperación autorizada sobre logs de sesión en vivo y durables, independientemente de la compactación.

| Paquete | Rol | clave ctx |
|---|---|---|
| [`session-query/`](session-query/README.md) | Define las lecturas de confianza, las consultas de relación y las operaciones de búsqueda | `ctx.sessionQuery` |
| [`session-query-sqlite/`](session-query-sqlite/README.md) | Implementa las consultas de sesión con búsqueda de texto completo de SQLite | `ctx.sessionQuery` |
| [`session-log-export/`](session-log-export/README.md) | Añade el comando Web `/export`, el estado de descarga de navegador compartido y el modal de resultado sobre el endpoint ZIP del Host | `ctx.sessionLogDownload` |
| [`tool-session-query/`](tool-session-query/README.md) | Expone al modelo las consultas de sesión autorizadas para el workspace | se registra en `ctx.tools` |

La referencia del subsistema — registros lógicos, lecturas acotadas, trazas, filtros, páginas de resultados — es [docs/subsystems/session-query.md](../../docs/subsystems/session-query.es.md).
