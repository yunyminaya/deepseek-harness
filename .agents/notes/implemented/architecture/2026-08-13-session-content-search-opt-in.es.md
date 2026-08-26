# Agent Note: La búsqueda de contenido de sesión se entrega opt-in con openAt never

Status: implemented

[English](2026-08-13-session-content-search-opt-in.md) | Español

## Problema

Los bundles entregados montaban el provider de consulta de sesión SQLite con el índice de texto completo activo (`openAt: first-search`), de modo que cada despliegue por defecto llevaba un índice FTS derivado y la barra lateral web ofrecía búsqueda de contenido. Que un despliegue quiera ese índice — su importación de node:sqlite, la reconciliación de fuentes por búsqueda y el almacenamiento derivado — es una decisión del despliegue, y el producto por defecto se entrega sin él; las herramientas de búsqueda orientadas al modelo ya eran opt-in y no estaban montadas (la [decisión de no entregar por defecto](../feature/2026-08-02-session-search-not-shipped-default.es.md)).

Apagar la capacidad desmontando la fila del plugin no es viable. `ApiProxyService` declara `sessionQuery` como inyección requerida, así que sin el provider todo el gateway de API del host permanece sin cargar y la GUI web nunca arranca. La exportación del registro de sesión traza los descendientes de subagente a través de `ctx.sessionQuery.traceSession`, y una bifurcación de subagente resuelve su Workspace mediante el mismo trazado de linaje — ambas necesitarían guardas de servicio opcional además de una fuente de linaje de reemplazo, triplicando aproximadamente la superficie de cambio y perdiendo lecturas exactas en todas partes.

## Decisión

La búsqueda de contenido se fuerza desactivada en el provider. `openAt: 'never'` es una tercera fase de apertura en `@deepseek-ai/dsh-session-query-sqlite`: `searchSessions` y `searchEvents` fallan con el código tipado `SESSION_QUERY_SEARCH_DISABLED` antes de cualquier normalización de la petición, node:sqlite nunca se importa ni se abre, y no se ejecuta observación ni reconciliación de fuentes. Toda lectura, filtro y trazado exacto de `ctx.sessionQuery` heredado sigue funcionando, de modo que la exportación de sesiones, la herencia de Workspace en las bifurcaciones y las lecturas de títulos no se ven afectados.

`SESSION_QUERY_SEARCH_DISABLED` se une a la taxonomía cerrada `SessionQueryErrorCode`, y el límite de servicio `tool-session-query` lo asigna al mensaje seguro para el modelo `session search is disabled in this deployment`.

El bundle base fija `openAt: never` en la fila `session-query-sqlite` y la reafirmación del bundle web lo mantiene; habilitar la búsqueda de contenido es una sobrescritura de `openAt` de una línea (`first-search` o `startup`) en una capa de parches posterior, normalmente con un `path` duradero. El endpoint `session.search` del host reporta el fallo del provider por su ruta de error existente, y la barra lateral web conserva su degradación diseñada: coincidencia local de títulos y workspace más el aviso de búsqueda de contenido no disponible. La spec de compatibilidad del CLI fija las filas `openAt: never` entregadas, mientras que el andamiaje e2e web mantiene la búsqueda de contenido habilitada — sus escenarios de sesión sembrada navegan por búsqueda de contenido, y esas ejecuciones son la cobertura ensamblada para la ruta opt-in.

## Alternativas consideradas

- **Desmontar la fila del plugin** (`disabled: true` en el parche base): rechazado — la inyección requerida `sessionQuery` del api-gateway mantiene toda la API del host sin cargar, y hacer esa inyección opcional obliga a guardas más un respaldo de linaje por recorrido de cabeceras en la exportación de sesiones y en la resolución de bifurcaciones.
- **Desactivar en los consumidores** (el endpoint `session.search` del host o la barra lateral): rechazado — la aplicación de la norma pertenece a la operación que toma la decisión; las herramientas modelo opt-in o cualquier otro consumidor seguirían llegando al índice.
- **Un booleano separado junto a `openAt`**: rechazado — la fase de apertura ya es dueña de cuándo arranca SQLite; `never` extiende el mismo eje en lugar de añadir un segundo mando que pueda contradecirlo.

## Consecuencias

- Los despliegues por defecto no ejecutan ningún índice derivado: sin importación de node:sqlite ni aviso de arranque de SQLite experimental, sin trabajo de reconciliación, sin base de datos derivada en disco. La búsqueda en la barra lateral solo coincide con títulos de sesión y nombres de workspace.
- Los fallos de búsqueda bajo el valor por defecto son tipados y estables en lugar de accidentales, de modo que los llamadores distinguen una decisión de despliegue de una avería del índice (`SESSION_QUERY_INDEX_FAILED`).
- Rehabilitar la búsqueda de contenido es configuración por despliegue, no un cambio de código, y restaura el comportamiento FTS completo sin cambios.
- Las composiciones que montan las herramientas de búsqueda sin sobrescribir `openAt` reciben el mensaje de desactivación seguro para el modelo en cada llamada de búsqueda; habilitar las herramientas implica habilitar el índice.
