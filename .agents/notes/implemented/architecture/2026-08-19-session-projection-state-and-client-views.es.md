# Agent Note: Separar el estado de proyección de sesión de las vistas del cliente

Status: implemented

[English](2026-08-19-session-projection-state-and-client-views.md) | Español

## Problema

El registro de proyecciones persistía el estado de fold interno de cada unidad sin un schema de runtime, mientras que `SessionProjectionMap` describía el valor de cliente devuelto por `view`. Esto dejaba el estado restaurado sin validar y hacía que la misma tabla de tipos pareciera describir dos valores que pueden diferir. Los consumidores del host necesitaban además el estado fold actual sin serializar cada vista de cliente registrada ni exponer estado solo interno a través del protocolo del cliente.

## Decisión

`SessionProjectionStateMap` es la tabla extensible por merge para los estados de fold del host. Toda clave de `ProjectionDefinition` pertenece a esta tabla y suministra un `stateSchema`; las filas cacheadas se validan antes de sembrar un fold. `SessionProjectionMap` conserva su significado y nombre existentes como la única tabla de valores completos visibles para el cliente, preservando las estructuras de datos de cliente existentes como `title: string | null`.

Una unidad cuya clave también aparece en `SessionProjectionMap` suministra `wire.viewSchema` y `wire.view`. El estado de toda unidad se checkpointea — visible para el cliente y solo del host por igual; el opt-in `persist` ha desaparecido, así que ninguna unidad puede omitir silenciosamente la caché durable. Las APIs de instantáneas devuelven solo `SessionProjectionMap`, de modo que los estados internos no pueden entrar en los payloads de API. El código del host lee un estado actual a través de `stateOf(session, key)`; la referencia devuelta es prestada y no debe mutarse.

## Consecuencias

El estado de proyección y los valores de cliente se tipan y validan de forma independiente sin introducir un segundo vocabulario DTO de cliente. Una unidad puede exponer un valor de cliente compacto o que preserva la compatibilidad mientras retiene un estado de host más rico. Un estado cacheado malformado no puede sembrar `viewCheckpoint`; la restauración rechaza el estado malformado y el respaldo de lectura completa existente de la caché lo reconstruye desde el log. Los consumidores del host pueden reemplazar los escaneos privados de log por el mismo fold incremental que usan los carriers.

La [propuesta original de proyección de sesión](../../proposed/architecture/2026-07-27-session-projection-and-command-log.es.md) registra ahora esta separación. Las decisiones anteriores de [proyección de identidad de subagente](2026-08-06-subagent-list-identity-projection.es.md) y [uso de tokens proyectado](2026-07-29-projected-token-usage-and-request-context.es.md) siguen vigentes; sus folds de dominio se mueven a la tabla de estados sin cambiar sus valores visibles para el usuario.

## Alternativas consideradas

- **Renombrar el mapa existente a tabla de estados e introducir un nuevo mapa de cliente** — rechazado porque cambia el nombre de tipo de cliente establecido e invita a migraciones innecesarias de payload de cliente.
- **Conservar una sola tabla para estado y valores de cliente** — rechazado porque un estado de fold más rico y un valor de cliente que preserva la compatibilidad no pueden entonces representarse con precisión.
- **Persistencia opt-in para unidades solo del host** — rechazado: un flag `persist` deja que una unidad omita silenciosamente la caché durable, y el ahorro (una fila pequeña por sesión) nunca justifica la asimetría ni la confusión de stateVersion que invita. El estado de toda unidad se checkpointea uniformemente.
- **Devolver estado copiado desde `stateOf`** — rechazado porque clonar cada lectura del host añade trabajo sin proteger ningún límite; el método documenta una obligación de referencia prestada de solo lectura para los llamadores tipados del mismo proceso.
