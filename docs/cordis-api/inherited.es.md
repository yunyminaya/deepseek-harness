<!-- El archivo en inglés lo genera scripts/gen-cordis-catalog.ts; este archivo en español es la contraparte revisada, mantenida mediante el emparejamiento bilingüe.
     Para actualizarlo, ejecuta primero `pnpm run gen-cordis-catalog` para regenerar el inglés y, a continuación, actualiza este archivo y ejecuta `pnpm run verify-translation-pairing --write docs/cordis-api/inherited.md` para volver a registrar el par. -->

# API de Cordis heredada

[English](inherited.md) | Español

Los miembros de `ctx` y los eventos del framework que todo plugin ve más allá del nivel del harness: fuente vendored fijada ([política de vendoring](../../vendor/README.md)), resumida con concisión para que las páginas del harness sigan centradas en el vocabulario propio del repositorio. Las API detalladas de Context, Fiber, Registry y Service se generan en [context.md](context.es.md), [fiber.md](fiber.es.md), [registry.md](registry.es.md) y [service.md](service.es.md); los métodos de despacho de eventos, en [events.md](events.es.md).

Este archivo se GENERA a partir del código fuente (`scripts/gen-cordis-catalog.ts`) y se verifica que esté fresco con `pnpm run verify-cordis-catalog` (parte de `doc-sync`) — no lo edites a mano. Los bloques de firmas usan un fence `ts cordis-catalog` e incluyen el JSDoc original de la fuente inmediatamente antes de cada evento o método de servicio. doc-typecheck omite estos fragmentos de declaración desnudos; los nombres de tipo de una firma enlazan a la página que los documenta.

## Miembros de `ctx` heredados (núcleo de cordis + loader/hmr/timer)

- `ctx.on / ctx.once` — Registra un listener de eventos (disposable). ([`vendor/cordis/src/events.ts:34`](../../vendor/cordis/src/events.ts))
- `ctx.emit / ctx.parallel / ctx.serial / ctx.bail / ctx.waterfall` — Despacha un evento (síncrono / con espera / primera devolución / cadena de cortocircuito). ([`vendor/cordis/src/events.ts:34`](../../vendor/cordis/src/events.ts))
- `ctx.plugin / ctx.inject` — Carga un plugin / declara los servicios requeridos. ([`vendor/cordis/src/registry.ts:164`](../../vendor/cordis/src/registry.ts))
- `ctx.effect` — Registra un efecto secundario disposable ligado al fiber. ([`vendor/cordis/src/fiber.ts:9`](../../vendor/cordis/src/fiber.ts))
- `ctx.get / ctx.set / ctx.provide / ctx.accessor / ctx.mixin` — Acceso y enlace de bajo nivel al almacén de servicios. ([`vendor/cordis/src/reflect.ts:7`](../../vendor/cordis/src/reflect.ts))
- `ctx.extend / ctx.isolate / ctx.intercept` — Deriva un contexto hijo (servicios con ámbito / aislamiento / intercepción). ([`vendor/cordis/src/context.ts:42`](../../vendor/cordis/src/context.ts))
- `ctx.root / ctx.scope / ctx.fiber / ctx.registry / ctx.reflect / ctx.events / ctx.logger` — Manejadores ambientales sobre el grafo de contextos en ejecución. ([`vendor/cordis/src/context.ts:16`](../../vendor/cordis/src/context.ts))
- `ctx.timer (+ interval / timeout / throttle / debounce)` — Helpers de temporizador disposable. La clave `timer` se provee en runtime; los cuatro helpers admitidos se mezclan directamente en ctx (declarados mediante Pick). ([`vendor/timer/src/index.ts:4`](../../vendor/timer/src/index.ts))
- `ctx.loader` — El Loader de configuración que arrancó la app (presente bajo el loader). ([`vendor/loader/src/index.ts:30`](../../vendor/loader/src/index.ts))
- `ctx.hmr` — El watcher de hot module reload (presente bajo el plugin hmr). ([`vendor/hmr/src/index.ts:15`](../../vendor/hmr/src/index.ts))

## Eventos heredados (núcleo de cordis + loader/hmr/timer)

- `internal/plugin` — Se creó un fiber de plugin. ([`vendor/cordis/src/events.ts:328`](../../vendor/cordis/src/events.ts))
- `internal/status` — Un fiber cambió de estado de ciclo de vida. ([`vendor/cordis/src/events.ts:330`](../../vendor/cordis/src/events.ts))
- `internal/service` — Hook de intercepción para un enlace de servicio (sin productor en el núcleo). ([`vendor/cordis/src/events.ts:332`](../../vendor/cordis/src/events.ts))
- `internal/update` — Waterfall (cascada de eventos): se está aplicando una actualización de configuración de un fiber. ([`vendor/cordis/src/events.ts:334`](../../vendor/cordis/src/events.ts))
- `internal/get` — Waterfall: se está leyendo un servicio del almacén. ([`vendor/cordis/src/events.ts:336`](../../vendor/cordis/src/events.ts))
- `internal/set` — Waterfall: se está escribiendo un servicio en el almacén. ([`vendor/cordis/src/events.ts:338`](../../vendor/cordis/src/events.ts))
- `internal/listener` — Se registró un listener. ([`vendor/cordis/src/events.ts:340`](../../vendor/cordis/src/events.ts))
- `internal/dispatch` — Se está despachando un evento a los listeners. ([`vendor/cordis/src/events.ts:342`](../../vendor/cordis/src/events.ts))
- `hmr/change` — Un archivo fuente vigilado cambió en disco. ([`vendor/hmr/src/index.ts:20`](../../vendor/hmr/src/index.ts))
- `hmr/reload` — Los plugins se están recargando después de un cambio. ([`vendor/hmr/src/index.ts:21`](../../vendor/hmr/src/index.ts))
- `exit` — El proceso está saliendo por una señal. ([`vendor/loader/src/index.ts:23`](../../vendor/loader/src/index.ts))
- `loader/config-update` — El árbol de configuración del loader cambió. ([`vendor/loader/src/index.ts:24`](../../vendor/loader/src/index.ts))
- `loader/entry-init` — Se está inicializando una entrada de configuración. ([`vendor/loader/src/index.ts:25`](../../vendor/loader/src/index.ts))
- `loader/partial-dispose` — Una entrada se está liberando parcialmente al recargar. ([`vendor/loader/src/index.ts:26`](../../vendor/loader/src/index.ts))
- `loader/patch-context` — Se está parcheando un contexto durante una recarga. ([`vendor/loader/src/index.ts:27`](../../vendor/loader/src/index.ts))
