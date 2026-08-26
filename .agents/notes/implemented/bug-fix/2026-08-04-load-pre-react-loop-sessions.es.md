# Agent Note: Cargar sesiones del formato pre-react-loop

Status: implemented

English | [中文](2026-08-04-load-pre-react-loop-sessions.zh.md)

## Problema

La simplificación react-loop cambió eventos duraderos reteniendo `SESSION_FORMAT_VERSION` 0. Sesiones almacenadas desde la base del cambio contienen `steering/message` y `turn/start.trigger`; sus razones de terminal también usan el `aborted` grueso, un `disposed` separado y dos payloads de error más viejos. Los invariantes actuales de superficie y turno no pueden repetir esos registros directamente.

El inbox duradero nuevo no es parte de este problema de compatibilidad. La base emitía notificaciones de inbox locales-al-proceso pero ningún evento de sesión `agent/inbox/*`, así que repetir historial viejo como trabajo pendiente resucitaría prompts ya reclamados o descartados.

## Decisión

`PersistenceCoordinator` reconoce las formas exactas pre-react-loop tras la decodificación del backend y las proyecta a la vista de lectura actual. Elimina el `turn/start.trigger` obsoleto, convierte `steering/message` al mismo `user/message` identificado, mapea hechos de fallo viejos al error estructurado actual, pliega `disposed` en un turno abortado con la causa `disposed`, y representa los registros gruesos aborted con la causa solo-persistencia `{ kind: 'legacy' }` porque su llamador no está disponible.

El coordinador aplica la proyección a `load`, `inspect`, adopción, comparación de prefijo HMR y `readFrom`. Un `readFrom` con capacidad de seek normalmente lee solo su sufijo; cuando ese sufijo contiene un evento legacy que necesita una identidad de reemplazo anterior, el coordinador carga y normaliza el prefijo completo antes de devolver el rango de seq pedido.

El importador no sintetiza empalmes de inbox. Un agente pre-react-loop resumido arranca con las listas pendientes vacías, igualando la incapacidad del runtime base de persistir trabajo pendiente de inbox. El artefacto almacenado permanece append-only y los eventos posteriores usan el formato actual.

## Alternativas consideradas

**Tratar los registros de la misma versión como no soportados.** Esto sigue el defecto pre-release pero vara sesiones producidas por la base del PR aunque el contenido de steering eliminado y los hechos terminales tienen mapeos completos.

**Repetir las notificaciones viejas de inbox como empalmes duraderos.** Esas notificaciones no eran eventos de sesión y no aportan una instantánea fiable de estado pendiente. Inferir inserciones sin cada claim y descarte re-corre trabajo consumido.

**Asignar los registros gruesos aborted a un llamador existente.** Mapearlos a `user`, `parent` o `hook` inventaría un llamador que el registro viejo no nombró. Una causa `legacy` dedicada conserva la clasificación de parada sin hacer una afirmación de auditoría falsa.

**Reescribir los registros JSONL y SQLite almacenados.** Una reescritura violaría el contrato append-only y exigiría maquinaria de migración atómica específica por backend para un límite de compatibilidad de lectura.

## Consecuencias

Las sesiones escritas en el formato base del refactor resumen a través del AgentLoop actual con su contenido de steering, fronteras de turno, hechos de error y clasificación de parada intactos. El contrato compartido del coordinador cubre `load`/`inspect`/`readFrom` en memoria, JSONL y SQLite, incluyendo el fallback de sufijo SQLite; un resume de Agent JSONL ensamblado verifica que la transcripción histórica es visible mientras ambas listas nuevas de inbox arrancan vacías.

Esta excepción soporta el formato base, no formatos intermedios producidos durante el desarrollo del refactor. En particular, no define migración para payloads experimentales anteriores `agent/inbox/spliced`. El reconocimiento de forma exacta mantiene los registros malformados con apariencia actual en su ruta de rechazo en vez de adivinarlos a validez.

## Relacionado

- [Cargar sesiones persistidas antes de la identidad de mensaje](2026-07-28-load-pre-identity-session-messages.es.md) — posee las identidades deterministas y el límite general de importación de solo-lectura para otro cambio de formato de la misma versión.
- [Persistencia de sesión como servicio abstracto](../architecture/2026-06-14-session-persistence.es.md) — posee el almacenamiento backend append-only y el resume.
