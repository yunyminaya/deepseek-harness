# AGENTS.md — Paquetes de Schedule

[English](AGENTS.md) | Español

Estas reglas complementan las instrucciones del repositorio y de los paquetes para `packages/schedule/*`.

- El flujo versionado `schedule/change` de la Session propietaria es el único estado durable de Schedule. Los folds validan cada frontera JSON durable y derivan los registros activos; los temporizadores, los waiters inactivos y los valores de herramientas siguen siendo proyecciones desechables.
- Una Session normal hace fold de su log completo. Una bifurcación deriva el estado activo de Schedule solo de los eventos desde `SessionHeader.seedLength` en adelante; nunca hereda un recordatorio activo de la Session padre.
- Toda operación de gestión de Schedule que lee o decide desde el fold espera primero `ctx.sessions.flush(session)`. Create y un delete real esperan una segunda barrera después del append; una barrera fallida devuelve el resultado estable de incertidumbre en lugar de inferir la durabilidad del log en vivo.
- Los dueños de runtime se adjuntan solo a root Agents en vivo futuros mientras el plugin está cargado. No escanean Sessions persistidas, no adoptan roots ya publicados, no despiertan Sessions frías, no registran herramientas globales ni borran registros durables durante el teardown.
- El manejo del trabajo debido re-comprueba el reloj de pared y el dueño en vivo exacto, reclama la fase de mantenimiento inactiva a través del seam público del Agent, construye el encuadre escapado completo antes de `followup()`, añade el dispatch solo después de que el enqueue síncrono devuelva, libera el mantenimiento y luego espera la durabilidad. Un fallo síncrono de encuadre/enqueue no añade ningún dispatch; un fallo posterior del modelo no lo revierte.
- La aritmética de reglas y la lógica de transición durable siguen siendo puras y deterministas. La producción usa el reloj de pared de la plataforma y temporizadores segmentados; los tests aportan muestras explícitas o temporizadores falsos sin añadir un servicio de reloj de producción.
