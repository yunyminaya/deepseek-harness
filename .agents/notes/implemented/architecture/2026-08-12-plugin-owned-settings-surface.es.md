# Agent Note: Superficie de ajustes propiedad del plugin

Status: implemented

[English](2026-08-12-plugin-owned-settings-surface.md) | [中文](2026-08-12-plugin-owned-settings-surface.zh.md) | Español

## Problema

Un plugin que registraba un namespace de ajustes no podía llegar a la página de configuración del navegador, y las dos compuertas que lo impedían vivían en este repositorio.

`packages/host/apiproxy` tenía dos listas de namespaces codificadas. `settings.describe` filtraba su respuesta a través de ellas y toda escritura las comprobaba primero, de modo que un namespace fuera de ellas respondía `settings-not-exposed` incluso cuando su dueño lo había registrado. Añadir un plugin a la página de configuración significaba por tanto editar un paquete que el autor del plugin no es dueño.

La sección de configuración de plugins renderizaba una lista no ordenada de las tarjetas registradas en `settings.plugin.item`. Una tarjeta llevaba un `id` opaco, nunca el namespace que editaba, de modo que la sección no podía saber qué namespaces servidos ya tenían un hogar. Eso dejaba toda pregunta sobre «quién renderiza este namespace» sin respuesta desde el registro que la sección podía ver.

Juntas, las dos significaban que un plugin escrito por el usuario solo era configurable editando `settings.yaml` a mano. La [nota de configuración web de plugins](../feature/2026-08-10-web-plugin-configuration.es.md) registró la allowlist como deliberada, y la [nota de límites del plano de configuración](2026-07-30-config-plane-boundaries.es.md) ató la configurabilidad web a la pertenencia al directorio de providers configurables. Ambas conclusiones bloqueaban exactamente a los autores de plugins para los que se construyó el seam general.

## Decisión

**Registrar es exponer.** El api-proxy sirve todo namespace que `ctx.settings.describe()` devuelve y no compuerta escritura alguna. `WEB_SETTINGS_NAMESPACES`, `PRODUCT_SETTINGS_NAMESPACES`, la unión con `ctx.llm.listConfigurableProviders()` y el código de error `settings-not-exposed` desaparecen. Un nombre al que ningún registro responde — desconocido, o malformado y por tanto incapaz de dirigirse a uno — se pliega en el propio `settings-rejected` del seam, de modo que el proxy no contribuye límite ni vocabulario propio.

**El seam de ajustes queda intacto.** Qué cliente puede leer un namespace, y qué página lo renderiza, son hechos sobre consumidores; una Service Definition que llevara cualquiera de los dos dejaría que un Consumer dictara su contrato. `SettingsRegisterOptions` no gana nada.

**`settings.plugin.item` está claveado por el namespace de ajustes.** El slot pasó de `list` a `keyed`, siendo la clave el namespace que edita la tarjeta, siguiendo el precedente de `tool.call.toolview` donde cada plugin de herramienta registra su renderizador bajo el nombre de la herramienta. Una tarjeta declara `key`, no `id`/`order`. El slot lo declara la pestaña `configurable` de la sección de Plugins, que es dueña de la lista de tarjetas.

**La pestaña conduce el despacho desde los namespaces servidos.** Deriva el conjunto servido actual de `ctx.settingsScope.describe()` y sigue ese espejo de ajustes compartido, mientras que su propio listener sigue el registro de slots de tarjetas. Despacha una clave por namespace servido. Lo que renderiza es la intersección de dos registros — namespaces que registró un plugin en vivo del Host, y tarjetas registradas bajo esas claves — calculada en el controlador de la pestaña a partir del registro de slots (`ctx.slots.entries`, `ctx.slots.subscribe`) y de la respuesta del espejo. La posterior [decisión del espejo de describe de ajustes](2026-08-17-settings-describe-mirror.es.md) es dueña de la lectura en todo el navegador y del ciclo de vida de invalidación.

El claveado hace de la ausencia la señal, y eso es lo que elimina la contabilidad que la forma anterior necesitaba. Un namespace que otra superficie es dueña (`ui-theme`, `permission`, `llm-*`, `agent-presets`) no tiene tarjeta bajo su clave, de modo que no renderiza nada sin declarar nada en ningún sitio. Una tarjeta cuyo namespace este despliegue no sirve jamás se despacha, lo que también arregla el viejo defecto de estado vacío: la pestaña contaba tarjetas registradas, incluidas las que no renderizaban nada, de modo que un despliegue que no exponía ninguna mostraba una lista vacía en lugar de su línea vacía.

**Nada renderiza un formulario que no se le dio.** La pestaña no suministra tarjeta de fallback. La mitad de navegador de un plugin es dueña de su tarjeta por completo — chrome, controles y redacción — que es lo que la opción `fallback` del slot habría reemplazado por un formulario renderizado por schema inverso.

## Qué protegía la allowlist

La compuerta sí mantuvo una cosa fuera del wire, y este note lo declara con claridad porque la decisión debe sobrevivir a la versión exacta: un namespace registrado que la lista no nombraba jamás llegó a ver sus valores resueltos, `base` o `user` llegar al navegador. La página de inventario de plugins no es un sustituto — `PluginInventoryEntry` lleva `entryId`, `moduleName`, `enabled` y `fiberPhase`, y su fila de «configuration» renderiza una etiqueta habilitado/deshabilitado, nunca un valor almacenado.

Lo que la compuerta no era es el límite que su posición sugería. Todo método de `settings.*` está en `PRIVILEGED_METHODS` (`packages/client/connection`), de modo que una solicitud de no-loopback o cross-origin se rechaza con 403 antes de llegar a este código; los campos `role('secret')` se eliminan estructuralmente de toda capa de toda respuesta; y el documento que el plano edita es el `settings.yaml` del propio usuario, que la misma página de ajustes ofrece abrir. Las escrituras que no bloqueaba eran también las consecuentes: `permission` (que puede ampliar el preset de aprobación) y `agent-presets` (que decide qué monta una sesión) ya se servían ambas.

Así que la exposición que este cambio añade de verdad, en este repositorio, es un namespace: `agent-default-model`, cuyos dos campos nombran un provider y un modelo y que ninguna mitad de navegador renderiza. Un namespace futuro cuyos valores genuinamente no deban cruzar el wire se responde por campo con `role('secret')` — más fino que un conmutador de namespace, y ya aplicado.

## Alternativas consideradas

**Una declaración en `settings.register()`** (`client: { surface: 'plugin-config' | 'custom', title, description }`), que el comentario eliminado de `WEB_SETTINGS_NAMESPACES` nombraba como la dirección prevista. Mantiene el registro fuera del transporte por defecto y permite que un autor de plugin se auto-sirva en una línea. Rechazado porque `surface` es vocabulario de página de navegador y `title`/`description` son presentación: una Service Definition que los lleva es un seam moldeado por un Consumer. Su propiedad de fallo-cerrado vale además menos de lo que parece — ver qué protegía la allowlist, arriba.

**Un catálogo de exposición separado**, un registro propio al que los plugins se unen junto a su registro de ajustes, generalizando `ctx.llm.registerConfigurableProviders()`. Rechazado porque hace que un hecho requiera dos registros que pueden derivar: registrar un namespace y olvidar la entrada de catálogo produce una sección que nada puede editar, sin compuerta capaz de ver el error.

**Un campo `Config` de deny-list en el api-proxy**, para que un despliegue pudiera retener un namespace. Rechazado por no tener consumidor: todo namespace actualmente registrado es uno que un usuario puede editar, y un campo genuinamente sensible se responde por campo con `role('secret')`, que es el instrumento más fino. Un conmutador a nivel de namespace inventado antes de su primer uso es la opción especulativa que las reglas de paquete prohíben.

**Una tarjeta genérica impulsada por schema como `fallback` del slot**, de modo que un plugin sin mitad de navegador recibiera igualmente un formulario de `schema.toJSON()` (schemastery ya lleva `description`, `role`, `min`/`max`/`step` y los serializa). Rechazado porque los plugins de cliente cargan en runtime desde entries de Loader montadas, de modo que un autor de plugin puede publicar una tarjeta real, y un formulario renderizado por inversa ya fue juzgado peor que uno escrito a mano para la página de Models. La opción `fallback` sigue disponible sin cambio de contrato si ese juicio cambia.

**Un registro de reivindicaciones del lado cliente**, donde cada superficie dueña de un namespace lo declara para que una tarjeta genérica sepa qué está ya cubierto. Rechazado junto con la tarjeta genérica: el despacho claveado ya hace que una clave no reivindicada no renderice nada, de modo que el registro repetiría lo que dice el registro de slots.

**Mantener el slot de lista y añadir un campo de namespace a sus opciones.** Rechazado porque la sección seguiría enumerando entries en lugar de namespaces, conservando el defecto de estado vacío y dejando que una tarjeta para un plugin no compuesto se suprimiera a sí misma.

## Consecuencias

Un plugin distribuido fuera de este repositorio es configurable desde la página de ajustes sin cambio aquí: registra su namespace en el Host y su tarjeta bajo esa clave en el navegador, y la sección empareja los dos. Las tarjetas aparecen ahora en el orden de registro de tarjetas en lugar de por un `order` asignado a mano. Eso es estable para las tarjetas que registra este paquete, que se instalan desde un generador, y **no** estable entre plugins: el orden de aplicación entre paquetes no está constreñido (`packages/client/AGENTS.md`), de modo que varias tarjetas externas pueden seguir reordenándose entre arranques. Ordenarlas necesita una clave explícita sobre la que la sección pueda ordenar, que el registro claveado no lleva hoy.

Diferido, y más grande que este cambio: el redactor devuelve un `role('secret')` alcanzable solo a través de una unión, intersección o transformación verbatim (su propio `TODO(settings-wire-redaction)`), y `schema.toJSON()` lleva el default de un secreto. Esa brecha precede a este cambio, pero servir todo namespace registrado amplía su radio de explosión de schemas auditados en este repositorio a cualquier schema de terceros, de modo que el wire debería rechazar un namespace del que no puede demostrar que sabe redactar. También diferida: una prueba de composición ensamblada de la capacidad titular — un plugin fixture montado por overlay cuya mitad de Host registra un namespace y cuya mitad `dsh.client` registra una tarjeta, afirmado de extremo a extremo. La cobertura actual demuestra cada mitad por separado; la salida sin cambios de las tarjetas publicadas no puede demostrar la ruta nueva.

La sección y sus tarjetas no añaden lecturas de `settings.describe`: ambas derivan del espejo en todo el navegador. Su invalidación es imprecisa en una dirección: el wire anuncia confirmaciones de documento y resets de conexión, no registros, de modo que un namespace registrado después de la respuesta actual del espejo se une en la siguiente confirmación o reconexión.

Quedan dos fricciones para un autor fuera de este repositorio, ambas registradas en el README de la sección. La mitad de navegador debe ser un paquete `dsh.client` construido en el formato de fábrica lazy-CJS del sistema de módulos de cliente, y el preset `clientBundle` que lo emite vive en `packages/client/tsdown.client.ts` en lugar de un paquete publicado. La compuerta de pureza de bundle prohíbe importar el chrome de tarjeta o el modelo de formulario por etapas de este paquete como valores, de modo que tal tarjeta reimplementa el staging y el vallado de revisión. Compartirlos significaría publicar el preset o declarar un slot hijo dentro de la tarjeta para que la sección suministre el chrome; ninguno está construido.
