# Agent Note: Tabla de inyección del índice estructurado (webserver/index-inject)

Status: implemented

[English](2026-08-19-web-index-injection-table.md) | Español

## Problema

El HTML de arranque del shell web necesita tres clases de inyección: el protocolo de arranque de client-modules (el script en línea de la cola de registro `__ModuleLoader__`, las etiquetas `<script src>` de precarga bloqueante del parser, el global de grafo `__DSH_BOOT__`) y el script de tema de primer pintado de ui-theme. El mecanismo antiguo eran transformaciones de cadena `webServer.tapIndex(html => html)`: cada registrante localizaba `<head>`/`<body>` con regex y empalmaba HTML por su cuenta. El despliegue con worker estático (la página es un artefacto de compilación; el árbol del host corre en un Web Worker) no tiene ningún paso de servir HTML, así que el lado worker copiaba a mano los mismos datos en su payload `/__boot__` (`graph` + `theme` vía `ctx.get`), y el lado página re-implementaba lo que hacían los taps (un instalador de fachada, un aplicador de tema, un bucle de precarga) — una semántica de arranque, tres implementaciones.

## Decisión

Convertir la superficie de inyección en un evento sobre datos puros: el webserver declara el evento `webserver/index-inject` y la unión de filas `IndexInjection` (`global`/`script`/`script-src`/`style`/`html`, colocación `head|body`). Un plugin que quiera inyectar se suscribe y empuja filas; toda recolección (`collectIndexInjections()`) es una emisión nueva, así que los suscriptores leen estado vivo en el momento de la emisión (grafo de módulos, preferencia de tema — sin caducidad de re-registro), y una suscripción muere con su fibra.

Una tabla, dos renderizadores: la forma servida, `webServer.renderIndex(html)`, renderiza las filas en index.html de forma determinista (filas de head tras la etiqueta head de apertura, filas de body tras la etiqueta body de apertura; `<` escapado JSON en los valores globales, `src` escapado por atributo); el payload `/__boot__` de la forma worker es `{ injections }`, ejecutado fila a fila por un pequeño intérprete del lado página (fijar global / crear elemento script / cargar externo a través del `loadBundle` del túnel / montar estilo y markup). Las filas son datos JSON puros — esa es la disciplina de equivalencia entre ambos extremos.

`tapIndex`/`applyIndexTaps` sobreviven como escape hatch de HTML crudo, aplicados después del renderizado de filas; todo consumidor interno se movió al evento.

## Consecuencias

- client-modules y ui-theme ya no editan HTML con regex; el `readBootPayload` del worker (meter mano en servicios: `clientModules`, `settings`, constantes de tema a través de `loader.load`) se elimina; las re-implementaciones del lado página `installModuleLoaderFacade`, `applyBootTheme` y `PARSER_PRELOAD_IDS` se retiran.
- Orden: entre suscriptores, el orden de suscripción (igual que el antiguo orden de taps); dentro de un suscriptor, el orden de empuje — los propios módulos garantizan cola → precargas → global.
- El renderizado servido del global del manifest cambió de `window.__DSH_BOOT__ =` a `globalThis["__DSH_BOOT__"] =`; ninguna expectativa de instantánea confirmada lleva ese texto, así que ninguna necesitó re-grabación.
- Los nuevos inputs de arranque visibles para el modelo o la página extienden la unión de filas; no hay consumidores nuevos de taps.

## Alternativas consideradas

- **Conservar las funciones de tap y añadir un renderizador del lado worker que las re-ejecute sobre un documento falso** — rechazado: los taps son cierres opacos `html => html`, así que el worker no puede serializarlos ni reproducirlos sin meter una emulación de DOM en el camino de arranque.
- **Una tabla estilo registro (`registerInjection(row): dispose`)** — rechazado por los dos problemas que el evento disuelve: las filas caducaban contra el estado vivo (preferencia de tema, grafo de módulos) salvo que cada productor se re-registrara en cada cambio, y cada productor era dueño de un disposer más. La extracción por emisión lee estado fresco con limpieza con ámbito de fibra gratis.
- **Borrar `tapIndex` de plano** — rechazado: un escape hatch para transformaciones HTML crudas no cuesta nada mientras la tabla es joven, y las composiciones externas pueden tener transformaciones que ninguna clase de fila expresa todavía.
