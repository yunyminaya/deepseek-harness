# Agent Note: Los detalles web siguen el ciclo de vida de la sesión actual

[English](2026-07-29-web-details-session-lifecycle.md) | [中文](2026-07-29-web-details-session-lifecycle.zh.md) | Español

Estado: implementado

## Problema

La entrada de detalles tiene ámbito de sesión, pero su ancho de cuadrícula preferido tiene ámbito de raíz. Seleccionar otra sesión sustituía el contenido de los detalles sin cerrar esa preferencia de raíz, así que el nuevo propietario heredaba una geometría de visualización obsoleta. El Hero y otros estados no seleccionados no renderizan detalles con ámbito de sesión; necesitan una pista derivada de cero sin convertirse en falsos propietarios en la comparación.

## Decisión

`AppFrame` lee el id de la sesión actual y su indicador `blank` del resumen desde la proyección autoritativa de sesión. Registra el último id no en blanco seleccionado solo cuando esa sesión puede ser dueña de los detalles, así que el Hero y otros estados no seleccionados ni disparan el cierre ni reemplazan al último propietario de sesión; la pista renderizada de sus detalles se deriva como cero sin cambiar la preferencia almacenada. La primera sesión preserva la preferencia inicial del almacén de diseño, cuya [decisión archivada de visibilidad por defecto](../../archived/bug-fix/2026-07-30-web-details-default-closed.md) eligió cerrada; volver a la misma sesión restaura su ancho actual, y seleccionar otra sesión cierra la preferencia de detalles con ámbito de raíz mediante el almacén de diseño antes del paint. La selección de chat por sesión sigue siendo propiedad del almacén con ámbito de sesión descrito por el [estándar del sistema de slots](../architecture/2026-07-22-slot-type-chain-implementation.es.md).

El almacén de diseño es transitorio y comienza con los detalles cerrados. Ni lee ni escribe `localStorage`, así que recargar restaura el valor por defecto de la barra lateral y los detalles cerrados y no necesita ninguna excepción de línea base de sesión. El cierre y la reapertura manuales dentro de una misma sesión sin cambios conservan su comportamiento existente. El efecto de ciclo de vida no cambia ni el [flujo de Nueva Sesión propiedad del Workspace](../feature/2026-07-25-workspace-ui-product-flow.es.md), ni los borradores del composer, ni la navegación de sesión, ni el redimensionado por cadena de concesiones.

## Alternativas consideradas

**Cerrar los detalles en el manejador de clic de Nueva Sesión.** Rechazada porque una superficie no seleccionada no tiene detalles con ámbito de sesión y no debe mutar la geometría. El cierre pertenece a la comparación posterior entre dos propietarios de sesión definidos.

**Persistir la geometría del panel por sesión.** Rechazada porque el contrato del producto necesita eliminar el contexto obsoleto, no un nuevo mapa de anchos recordados. La geometría por sesión además reabriría los detalles al volver, en contra del comportamiento elegido de cerrar al salir.

**Preservar el diseño persistido después de que la línea base de sesión esté lista.** Rechazada porque duplica el ciclo de vida de arranque en un componente de presentación solo para validar el estado de visualización obsoleto. Los valores por defecto transitorios hacen determinista la recarga sin un indicador de preparación.

**Tratar todo cambio de la proyección actual como un cambio de sesión.** Rechazada porque la materialización de arranque, el Hero, el borrado de la selección y la invalidación no son transiciones entre dos propietarios de sesión.

## Consecuencias

Los detalles comienzan cerrados, incluso cuando se materializa la primera sesión. Una acción de apertura explícita usa el ancho por defecto del contrato. Cambiar a otra sesión olvida un ancho de detalles arrastrado porque el cierre escribe cero y la reapertura usa ese valor por defecto. Los estados no seleccionados derivan una pista renderizada de cero mientras dejan la geometría preferida sin cambios; volver a la misma sesión a través de uno de esos estados restaura su ancho. La recarga olvida la geometría de la barra lateral y restaura los detalles cerrados. La prueba de comportamiento del diseño cubre los valores por defecto iniciales, la primera materialización, los cambios de sesión directos y mediados por el Hero, la vuelta a la misma sesión y la ausencia de almacenamiento de diseño; el e2e de navegador sin clave conduce las mismas transiciones de propietario a través de la composición publicada mientras comprueba la pista de cuadrícula completa y los errores de navegador.
