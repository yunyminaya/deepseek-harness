# Agent Note: Simplificar la representación del log de sesión

Status: implemented

[English](2026-07-12-simplify-session-log-representation.md) | Español

## Problema

El log de sesión mantiene dos representaciones que cuestan más maquinaria de la que sus consumidores requieren: una superficie pseudo-enlazada y deltas de cabecera de request personalizados.

`SurfaceManager` almacena el mismo orden en un array, un mapa de seq y enlaces mutables `prev`/`next`. La producción nunca lee ninguno de los dos enlaces: el balance de emparejamiento de herramientas de compact usa saldos por corte almacenados en orden de superficie. El reemplazo ya usa `indexOf`, así que los enlaces no hacen constante su operación dominante. Un array de seq con búsqueda de reemplazo lineal tiene el mismo coste asintótico de reemplazo y una representación que validar.

El subsistema de cabecera de request implementa un codec de delta sistema/herramientas personalizado y una capa de decisión de transmisión aunque su contrato diga que los deltas son una optimización de codificación, no un requisito de reconstruibilidad. Conservar la instantánea completa inicial/reanudación en cada límite de instancia del loop, y luego escribir una `request/header` completa canónica cada vez que la cabecera ensamblada de esa instancia cambie, preserva el replay mientras elimina `SystemDelta`, `ToolsDelta`, el fallback de ida y vuelta y la variante durable `request/header-delta`. El vocabulario solo-de-codec desaparece con el codec, no porque sus brazos individuales fueran inválidos.

La implementación conserva los `sourceEventSeqs` de append y reemplazo, la seq de `tool/call` citada por los resultados reparados por crash y todas las variantes de `SessionStartSource`, porque esos campos tienen un papel de auditoría/intercepción que cero lectores actuales no revierte.

## Decisión

`SurfaceManager.nodes` es un `readonly number[]` de secuencias de eventos; la forma pública `SurfaceNode`, los enlaces de nodo y el mapa seq-a-nodo se eliminan. La señal interna de generación de reemplazo permanece. La lectura completa `foldSurface()` usada por session-query devuelve la misma representación de array de números más los metadatos de reemplazo sin hacer que el gestor incremental conserve historial. El balance de emparejamiento de herramientas y la compactación usan secuencias de eventos y posiciones de superficie; la caché de saldos por corte propiedad de compact no depende de enlaces de nodo.

Las cabeceras de request usan solo instantáneas completas canónicas. Los anclajes inicial y de reanudación siguen siendo instantáneas completas incluso cuando no cambian; un cambio dentro de la instancia añade otra `request/header` completa con razón `change`. El evento delta, los tipos del codec, los helpers diff/apply y la razón `fallback` solo-de-codec se eliminan. La reconstrucción del request selecciona la instantánea más reciente.

`SESSION_FORMAT_VERSION` permanece fijada en `0`, así que la validación de seed, append y carga de persistencia rechaza explícitamente los eventos antiguos v0 `request/header-delta` y las instantáneas completas que llevan la razón `fallback` eliminada. No hay fold de compatibilidad ni migración. Las pruebas de JSONL y SQLite fijan este límite de fallo-ruidoso, y el harness de instantáneas ACP representa los cambios legítimos a mitad de sesión como cabeceras completas fijadas y prompts completos legibles.

## Alternativas consideradas

**Conservar los nodos enlazados y los deltas compactos para un posible escalado.** Los enlaces podrían ayudar a una futura API de cursor, y los deltas pueden reducir los logs cuando grandes schemas de herramientas cambian en una pequeña cantidad. Ningún cursor publicado usa los enlaces, mientras que las instantáneas completas intercambian tamaño de disco por una corrección sustancialmente más simple. Si el volumen de cabeceras resulta material, se puede diseñar compresión o un esquema de delta canónico medido alrededor de trazas reales.

## Verificación

La cobertura unitaria fija el comportamiento de append/reemplazo de superficie ordenada, el emparejamiento de herramientas, la compactación, el plegado/registro de cabeceras completas, la reconstrucción del request y los invariantes de desarrollo. La validación de seed más las pruebas de carga de JSONL y SQLite rechazan el evento legado antes del replay. La suite ACP sin clave ejercita record, refresh, replay, fijación de cabecera cambiada y el fixture de cambio de modo sandbox en la nueva forma.

## Consecuencias

Las cabeceras completas aumentan el volumen del log, y la búsqueda de reemplazo lineal podría ser más lenta en superficies muy grandes. Los reemplazos ya eran lineales porque la implementación anterior llamaba a `indexOf`; los benchmarks se difieren hasta que trazas reales muestren que el array más simple es un cuello de botella. La versión de formato sigue siendo `0`, así que el rechazo explícito de eventos legados es una parte permanente del límite de formato pre-release. A cambio, el orden de superficie y el estado de cabecera de request tienen cada uno una representación, eliminando el mantenimiento de enlaces, los mapas, los brazos del codec, el fallback de ida y vuelta y la normalización de instantáneas consciente de deltas.
