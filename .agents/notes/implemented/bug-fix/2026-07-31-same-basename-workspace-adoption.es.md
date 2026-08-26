# Agent Note: Adopción de Workspaces con el mismo basename

Status: implemented

English | [中文](2026-07-31-same-basename-workspace-adoption.zh.md)

## Problema

Un Workspace se identifica por su id estable y su ruta canónica de directorio, mientras su título es metadato mutable de despliegue. El registro sin embargo rechazaba una ruta canónica nueva cuando su título derivado del basename igualaba a otro Workspace. Layouts de directorio comunes como `/a/xx` y `/b/xx` por tanto no podían coexistir en la Web UI, aunque el [diseño de dominio](../../proposed/architecture/2026-07-24-domain-kv-storage-and-workspace.es.md) ya permite títulos duplicados y cada operación de cliente dirige un Workspace por id.

## Decisión

`ctx.workspaceRegistry.create(path, title?)` trata la ruta canónica como la única clave de unicidad. Repetir la misma ruta sigue siendo idempotente y conserva el título registrado. Rutas canónicas distintas crean registros de Workspace distintos y pueden compartir título; cuando no se suministra título, cada registro sigue derivando su título de `basename(path)` sin sufijarlo ni reescribirlo.

La ruta de adopción `workspace.create({ path })` del Host hereda esa regla. El gestor de Workspaces, el picker, el árbol de agrupación, la selección, el renombrado, el borrado y la creación de Sesiones siguen usando `WorkspaceId`, así que etiquetas iguales ni fusionan registros ni redirigen una operación. La tarjeta de hover de la barra lateral expone cada ruta canónica cuando las etiquetas necesitan desambiguación.

El nombramiento explícito sigue siendo más estricto. `workspace.rename` sigue rechazando un título ya registrado, como describe el [nombramiento manual de Workspace](../feature/2026-07-25-session-list-browsing-and-manual-order.es.md). Esto evita que un usuario introduzca deliberadamente otra etiqueta ambigua mientras acepta colisiones impuestas por nombres de directorio existentes. La regla de adopción por ruta sustituye solo las cláusulas de conflicto-de-título en el [flujo producto de Workspace](../feature/2026-07-25-workspace-ui-product-flow.es.md) y el [picker nativo de directorios](../feature/2026-07-27-native-workspace-directory-picker.es.md).

El esquema duradero no cambia: los registros de Workspace ya almacenan id, ruta y título de forma independiente, el bootstrap puede derivar basenames iguales, y el arranque valida rutas duplicadas en vez de títulos.

## Verificación

Los tests del registro de Workspaces y de la API del Host crean dos directorios reales bajo padres distintos con el mismo segmento final y asertan ids, rutas y orden duradero distintos. El componente picker renderiza etiquetas iguales como entradas separadas keyed por id. El escenario keyless de browser Web adopta ambos directorios a través del flujo compuesto de directorios y observa dos Workspaces registrados y renderizados.

## Alternativas consideradas

**Conservar la unicidad de título y rechazar el segundo directorio.** Una etiqueta de despliegue seguiría siendo una clave de identidad accidental y los layouts ordinarios multi-raíz seguirían imposibles de registrar.

**Sufijar automáticamente los títulos en colisión.** Una etiqueta generada como `xx (2)` ya no sería el título derivado del directorio, necesitaría reglas estables de asignación a través de borrado y recarga, y añadiría estado solo para ocultar un error de identidad.

**Usar la ruta completa como título de todo Workspace.** Esto elimina la colisión pero hace la etiqueta primaria de navegación innecesariamente larga. La ruta completa sigue disponible en el detalle de hover mientras el basename conciso sigue siendo útil.

**Permitir colisiones también desde la operación explícita de renombrado.** El registro soporta ese estado, pero rename pide intencionalmente al usuario elegir un nombre de despliegue. Retener su respuesta de conflicto preserva el guard de nombramiento existente sin bloquear rutas seleccionadas por el filesystem.

## Consecuencias

Dos filas de Workspace pueden llevar el mismo título visible. Permanecen independientemente seleccionables y accionables porque los ids poseen la identidad; los usuarios pueden inspeccionar la ruta o renombrar cualquiera de las dos filas para desambiguarla. Un renombrado explícito no puede seleccionar el título actual de otra fila, incluyendo un título surgido de la adopción por mismo-basename. No se requiere migración de almacenamiento ni ruta de compatibilidad.
