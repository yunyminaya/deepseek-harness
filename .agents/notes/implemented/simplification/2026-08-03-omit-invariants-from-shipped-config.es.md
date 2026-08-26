# Agent Note: Omitir los invariantes de runtime de la configuración dsh entregada

Status: implemented

[English](2026-08-03-omit-invariants-from-shipped-config.md) | Español

## Problema

`@deepseek-ai/dsh-invariants` y los companions `./invariant` propiedad de cada paquete son diagnósticos de desarrollo opcionales. La TUI entregada montaba el servicio y cuatro companions con estado mientras que el árbol Web entregado los omitía, así que las dos superficies de producto tenían un coste y un comportamiento de fallo diagnósticos distintos. Un fallo de aserción relacional podía terminar una ejecución TUI ordinaria aunque la frontera de producto siempre activa siguiera siendo responsable de la validación de sesión y la historia inmutable.

## Decisión

Los árboles de configuración `dsh` entregados bajo `apps/cli/config/` no montan ni `@deepseek-ai/dsh-invariants` ni ningún companion `./invariant` propiedad de un paquete. El paquete del CLI por tanto no lleva dependencia directa del servicio de invariantes.

El soporte de invariantes sigue disponible para pruebas enfocadas, bundles de ejemplo, composiciones del SDK generado y despliegues personalizados que se adhieren explícitamente a los diagnósticos. La validación de sesión, las instantáneas, el congelado y la validación citada de eventos fuente siguen siempre activos y no dependen del servicio opcional, como define la [decisión de inmutabilidad propiedad de la fuente](../architecture/2026-06-11-dev-invariants-over-deep-readonly.es.md).

La prueba de config-dump del CLI construido comprueba ambas superficies entregadas y rechaza tanto la entrada del servicio como cualquier entrada `@deepseek-ai/dsh-*/invariant`.

## Alternativas consideradas

- **Montar el servicio con `enabled: false`.** Rechazada porque el árbol entregado y la dependencia del CLI seguirían cargando diagnósticos que no instalan ninguna comprobación.
- **Conservar el montaje solo para TUI.** Rechazada porque las superficies entregadas retendrían un comportamiento diagnóstico y de fallo distinto.
- **Eliminar el soporte de invariantes del repositorio.** Rechazada porque las comprobaciones propiedad de cada paquete siguen siendo útiles en pruebas, ejemplos, SDKs generados y composiciones explícitas de desarrollo; solo la configuración de producto por defecto queda fuera de alcance.

## Consecuencias

- Las ejecuciones TUI y Web ordinarias de `dsh` no instalan listeners de invariantes ni estado de traza y no pueden fallar a través de `InvariantError`.
- Las composiciones de desarrollo y personalizadas conservan acceso explícito al servicio de invariantes y a los companions.
- La ausencia en la configuración entregada se verifica desde la salida compuesta del CLI construido para ambas superficies.
- La integridad de sesión siempre activa permanece sin cambios.
