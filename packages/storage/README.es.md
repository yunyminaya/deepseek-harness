# storage/ — familia de almacenamiento no-sesión

[English](README.md) | Español

Esta familia persiste datos de aplicación distintos de los logs de eventos de sesión a través de backends con nombre y formas de datos tipadas.

| Paquete | Rol | clave ctx |
|---|---|---|
| [`storage/`](storage/README.md.es.md) | Conecta los backends registrados con las formas de datos tipadas | `ctx.storage` |
| [`storage-json/`](storage-json/README.md.es.md) | Almacena datos en archivos JSON | registra el backend `json` |
| [`storage-sqlite/`](storage-sqlite/README.md.es.md) | Almacena datos en SQLite | registra el backend `sqlite` |
| [`storage-domain/`](storage-domain/README.md.es.md) | Proporciona almacenamiento validado de registros de dominio | `ctx.storageDomain` |

Los consumidores usan una forma de datos en lugar de acceder directamente a un backend. La [decisión de almacenamiento de dominio](../../.agents/notes/proposed/architecture/2026-07-24-domain-kv-storage-and-workspace.es.md) registra el diseño de la familia.

La referencia del subsistema — el contrato del backend, `StorageForms`, `DomainSpec`/`Domain`, `domain/changed` — está en [docs/subsystems/storage.md](../../docs/subsystems/storage.es.md).
