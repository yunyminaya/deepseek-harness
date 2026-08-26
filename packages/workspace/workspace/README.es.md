# @deepseek-ai/dsh-workspace

[English](README.md) | [中文](README.zh.md) | Español

Registro de entidades de espacios de trabajo (`ctx.workspaceRegistry`) para DeepSeek Harness: registros durables de espacios de trabajo, orden estable de espacios de trabajo y un índice de sesiones candidatas de más reciente a menos reciente almacenado mediante la forma de datos de dominio. Los consumers ven la interfaz `Workspace`; la implementación de la entidad permanece privada al paquete.

El fundamento de entidad/almacenamiento vive en la [Agent Note de dominio](../../../.agents/notes/proposed/architecture/2026-07-24-domain-kv-storage-and-workspace.es.md); el bootstrap de solo cabeceras y el orden en la GUI viven en la [Agent Note del flujo de producto de Workspace UI](../../../.agents/notes/implemented/feature/2026-07-25-workspace-ui-product-flow.es.md).

## Forma

- `ctx.workspaceRegistry.create(path, title?)` — canoniza `path` vía `fs.realpath`, rechaza una ruta inexistente o que no sea directorio, crea como máximo un registro por ruta canónica y antepone un registro nuevo al orden durable de espacios de trabajo. Llamadas repetidas para esa ruta devuelven el espacio de trabajo existente sin cambiar su título; rutas diferentes pueden compartir un título de visualización.
- `ctx.workspaceRegistry.get(id)` / `list()` / `resolveByPath(path)` — búsquedas servidas desde caché. `list()` es síncrona y sigue el orden durable del registro; `resolveByPath` es asíncrona porque aplica la misma canonización `realpath` y rechaza una ruta ausente en lugar de crearla.
- `ctx.workspaceRegistry.insertBefore(id, before?)` — mueve un Workspace registrado dentro del orden durable del registro, al estilo insertBefore del DOM: antes del ancla, o al final cuando se omite el ancla. Un origen o un ancla ausentes del registro rechazan sin escribir; un auto-anclaje o un movimiento a la posición actual se resuelven sin escribir. La lista de ids devuelta es el orden confirmado completo.
- `ctx.workspaceRegistry.delete(id)` — elimina únicamente el registro del Workspace, su entrada de orden durable y su cuenta de sesiones. Los ids desconocidos devuelven `false`; un registro eliminado devuelve `true`. El directorio, los archivos del usuario, las Sessions vivas y los logs de sesión persistidos nunca se tocan, así que esas Sessions pasan a ser Ungrouped. Un fallo de escritura de tabla restaura el orden y la entidad publicados previamente.
- `Workspace.attachSession(id)` — valida una cabecera de sesión viva o persistida cuyo cwd sea el directorio del espacio de trabajo y antepone un id nuevo. Las sesiones desconocidas, los valores de cwd ausentes/no resolubles/no directorio y las discrepancias rechazan sin escribir. `detachSession` elimina únicamente la entrada del índice de candidatas.
- `Workspace.insertSessionBefore(id, before?)` — mueve una sesión contabilizada dentro del orden manual, al estilo insertBefore del DOM: antes del ancla, o al final cuando se omite el ancla. Una sesión o un ancla ausentes de la cuenta rechazan sin escribir; un movimiento a la posición actual se resuelve sin escribir. El orden de Workspace del registro nunca cambia.
- `ctx.workspaceRegistry.archiveSession(id)` / `archivedSessionIds` — el conjunto de archivado global del registro, superpuesto sobre la contabilidad del espacio de trabajo: una sesión archivada desaparece de las superficies de agrupación pero conserva su log de sesión y su slot `sessionIds`, así que un desarchivado futuro restaura su posición. Archivar acepta cualquier sesión viva o persistida (contabilizada o Ungrouped), se resuelve sin escribir para un id ya archivado y rechaza un id desconocido. El estado escrito antes de que existiera el campo se analiza con un conjunto vacío.
- `Workspace.sessionIds` — proyección síncrona de ids más cwd canónico de la membresía, en orden durable de candidatas. Las cabeceras ausentes, los valores de cwd inválidos y las discrepancias se filtran; la siguiente mutación del espacio de trabajo los poda. Un medio que indexe una sesión bajo dos espacios de trabajo, que reclame una ruta desde dos registros o que diverja del orden durable de espacios de trabajo rechaza en el arranque.
- `Workspace.status()` — comprobación de directorio sin caché, `'ok' | 'missing-dir'`; un directorio ausente nunca muta el registro.

`storageDomain` y `sessionPersistence` son dependencias de arranque obligatorias. Un par no disponible deja el plugin pendiente e impide confirmar un marcador vacío de inicialización. En el primer arranque correcto, el registro llama a `SessionPersistence.list()` y usa solo las cabeceras `id`, `cwd` y `createdAt` para agrupar los directorios históricos válidos y persistir el orden inicial; nunca lee los cuerpos de eventos. El marcador de inicialización se escribe al final, así que las escrituras parciales de bootstrap se reutilizan con seguridad tras un reinicio. Las sesiones posteriores de solo-cwd permanecen Ungrouped.

Crear y eliminar persisten un marcador explícito de mutación pendiente antes de que su registro y su orden puedan divergir. El arranque completa únicamente la mutación marcada y luego limpia el marcador; un desajuste de orden/tabla sin marcar sigue siendo corrupción inexplicada y falla de forma estridente. Eliminar y volver a registrar la misma ruta crea un id de Workspace fresco y no readopta automáticamente las Sessions retenidas.

## Experiencia del modelo

### Registros de espacios de trabajo y cuentas de sesiones

#### Lo que ve el modelo

Nada. `ctx.workspaceRegistry` sirve registros de espacios de trabajo solo a consumers del lado del host: el paquete no registra herramientas, no inyecta prompts y no escribe eventos de sesión, así que ningún campo de petición lleva jamás datos de este paquete.

#### Efecto de tokens

Cero tokens directos en cada petición.

#### Efecto de KV Cache

Independiente de las peticiones vivas: el paquete nunca toca un prefijo de petición, así que no puede invalidar la reutilización de la caché del provider.

## Limitaciones conocidas y trabajo diferido

- La eliminación de sesiones y la retirada destructiva de carpetas son capacidades separadas y ausentes; la eliminación del registro de Workspace nunca sustituye a ninguna de las dos ([decisión](../../../.agents/notes/implemented/feature/2026-07-27-workspace-registration-deletion.es.md)).
- El índice de cabeceras se refresca en el arranque y cuando attach debe resolver un id persistido sin caché; una eliminación o un daño del cwd causados por otro proceso se observan tras el siguiente refresco o reinicio.
