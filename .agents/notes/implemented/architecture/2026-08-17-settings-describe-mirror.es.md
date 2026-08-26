# Agent Note: Espejo de describe de ajustes

Status: implemented

[English](2026-08-17-settings-describe-mirror.md) | Español

## Problema

Un arranque web en frío emitía `settings.describe` quince veces en unos ~200 ms, y el recuento crecía en dos con cada plugin de cliente que era dueño de una preferencia. Dos mecanismos se apilaban: `SettingsScopeBinder.bind()` iniciaba una lectura de documento completo por ámbito vinculado (seis ámbitos en la composición del producto, más la pestaña del directorio de plugins, la puerta de bienvenida y la incorporación del onboarding de modelos), y `onConnected` emite `connection/reset` también en la PRIMERA conexión, de modo que cada uno de esos lectores releía inmediatamente la respuesta que había obtenido milisegundos antes. Cada lector llevaba además sus propias suscripciones de invalidación y su propia guarda estilo `refreshIfLoaded`, y quince lecturas independientes podían en principio aterrizar en quince revisiones de documento distintas.

## Decisión

**Un lector, muchas derivaciones.** `dsh-client-ui-settings` es dueño de `SettingsDescribeMirror`, el único lector de `settings.describe` en el navegador: un único almacén de instantáneas que contiene toda la respuesta, refrescado por las dos suscripciones del plugin dueño (`settings/document-updated`, `connection/reset`). Las llamadas concurrentes a `load()` se pliegan en la lectura en vuelo más a lo sumo una re-ejecución. El slot en vuelo es dueño de una ejecución antes de que su publicación de carga pueda reentrar síncronamente en `load()`, y luego se limpia dentro del propio try/finally de la ejecución en el mismo segmento síncrono que observa el flag de re-ejecución; un `.finally()` en la promesa devuelta se ejecutaría un microtask más tarde y dejaría que un refresco aterrizado en ese hueco marcara una re-ejecución que nadie lee.

`bind()` sigue devolviendo la cara `SettingsScope<T>` sin cambios, pero el controlador es ahora un selector sobre el espejo: sin ruta de lectura propia, las mismas reglas de decodificación y la cola de escritura conservada. Una escritura confirmada pliega su vista respondida de vuelta al espejo (`acceptView`), de modo que los ámbitos hermanos ven la nueva revisión sin re-leer; el pliegue invalida cualquier respuesta en vuelo más antigua, y una escritura anterior al primer documento retenido re-ejecuta esa lectura en lugar de publicar un documento parcial. Una escritura reciente fallida provoca una lectura de recuperación del espejo. Las superficies entre namespaces — la pestaña del directorio de plugins, la fila de permisos (su enum dinámico vive en el schema del namespace, que los ámbitos deliberadamente no llevan), la incorporación de modelos, la escribibilidad de la fila de agent-preset y `hasDocument` — consumen `ctx.settingsScope.describe()`, la cara compartida de lectura/pliegue (`getSnapshot`/`subscribe`/`ensure`/`acceptView`).

Esta decisión actualiza la mecánica de lectura e invalidación del navegador registrada por [Preferencias web respaldadas por el host](../bug-fix/2026-08-06-host-backed-web-preferences.es.md) y [superficie de ajustes propiedad del plugin](2026-08-12-plugin-owned-settings-surface.es.md), preservando sus decisiones de titularidad de preferencias y exposición de namespaces. También reemplaza la descripción de lectura directa de ajustes en [configuración de credenciales del primer arranque oficial de DeepSeek](../feature/2026-07-30-deepseek-onboarding-credential-setup.es.md); esa incorporación ahora deriva su mitad de ajustes de este espejo.

El presupuesto de arranque en frío queda fijado en dos lecturas por `apps/web/tests/startup-rpc-budget.e2e.ts`: la lectura ansiosa de bind del espejo, más la lectura de reset de la primera conexión, que se conserva deliberadamente — cierra la ventana donde un commit de documento aterriza entre la lectura HTTP ansiosa y la suscripción SSE y su invalidación se pierde. El objetivo original del plan de una lectura es inalcanzable sin aceptar esa ventana de invalidación perdida o retrasar la primera lectura hasta que el flujo SSE se abra.

## Alternativas consideradas

- **Single-flight compartido solo dentro de `bind()`** — deduplica las ráfagas concurrentes pero mantiene N lectores directos, N conjuntos de suscripciones y el sesgo de revisiones; los lectores fuera del binder (bienvenida, modelos, pestaña, permiso) no ganan nada. Rechazado por tratar el síntoma.
- **Incrustación en el payload de arranque** (el host incrusta la respuesta de describe en el arranque de la página) — ahorra la primera lectura pero añade una segunda ruta de adquisición con sus propias reglas de caducidad encima del espejo que igualmente seguiría necesitando. Diferido; compone con el espejo si alguna vez se quiere.
- **`settings.describe(ns)` por namespace** — reduce cada respuesta pero mantiene una lectura por consumidor, así que la dispersión y la tasa de crecimiento se mantienen. Rechazado.
- **Una lectura (sin re-lectura del primer reset)** — alcanzable solo aceptando la ventana de invalidación perdida entre la lectura HTTP ansiosa y la suscripción SSE, o retrasando la primera lectura hasta que el flujo se abra; ambos intercambian corrección o frescura del primer pintado por una petición de ida y vuelta. Rechazado en favor de las dos fijadas.

## Consecuencias

- El `settings.describe` de arranque pasó de 15 → 2, y un plugin nuevo dueño de preferencias añade cero lecturas.
- Toda superficie derivada muestra la misma revisión de documento en cualquier momento; las guardas por lector (`refreshWelcomeIfLoaded`, `refreshPermissionIfLoaded`, `refreshDocumentIfLoaded`) y sus suscripciones han desaparecido.
- El espejo se refresca en cada commit de documento sin importar el namespace, de modo que una edición externa de ajustes cuesta ahora una lectura en segundo plano incluso sin ninguna superficie de ajustes abierta — el precio de superficies que se abren ya frescas. Los filtros `ns !== spec.namespace` por namespace han desaparecido con las suscripciones por ámbito.
- `credentials.describe` (3 llamadas de arranque), `agentPreset.list` (2) y `llm.providers` son fuentes separadas y siguen siendo directas; el mismo patrón de espejo les encaja si alguna vez lo necesitan.
- Un llamador directo nuevo de `settings.describe` en código de cliente es una regresión de presupuesto; el mensaje de fallo del e2e dice que busques llamadores fuera de `ui-settings`.
