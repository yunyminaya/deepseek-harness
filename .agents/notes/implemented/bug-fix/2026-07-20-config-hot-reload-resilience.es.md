# Agent Note: Un hot-reload de configuración no debe matar ni degradar una aplicación en marcha

Status: implemented

[English](2026-07-20-config-hot-reload-resilience.md) | Español

## Problema

Una edición inválida de `cordis.yml` no debe matar a un agent (agente) en ejecución, pero conservar el proceso es insuficiente cuando una actualización de aspecto válido reemplaza parcialmente el árbol del Loader antes de que una entrada posterior falle. Quienes llaman también necesitan observar una actualización en vivo rechazada sin tratar el mismo error como un fallo de arranque no manejado. La configuración personal añade un segundo requisito: HMR (hot module replacement) debe observar un archivo exacto fuera de sus raíces de módulos, incluido un archivo o un directorio padre creado después del arranque.

## Decisión

Los plugins de ciclo de vida y de Loader de Cordis vendido proporcionan una transacción de configuración compensatoria y esperada, registrada como modificaciones locales 6, 8 y 9 en [vendor/README.md](../../../../vendor/README.md).

`Fiber.update()` devuelve su resultado del waterfall (cascada de eventos) de `internal/update`. La validación de configuración sigue siendo síncrona, mientras que la continuación por defecto devuelve la promesa del reinicio. Las actualizaciones de entradas del Loader pueden por tanto distinguir el fallo de validación, importación, aplicación y rollback del asentamiento exitoso del ciclo de vida. `EntryTree.await()` vuelve a comprobar los fibers con gate de servicio después de que las tareas del Loader se drenan y rechaza los fallos asentados; un fiber que espera un servicio ausente sigue siendo una entrada pendiente válida en lugar de hacer que el asentamiento se cuelgue.

El Loader importa un nombre de módulo cambiado antes de deshacer el fiber activo. La aplicación del candidato se espera; un fallo deshace los efectos del candidato y restaura el plugin o la configuración previos. La reconciliación de grupos inicia los candidatos en paralelo, espera cada resultado y restaura las entradas cambiadas, las adiciones, las eliminaciones y los movimientos antes de rechazar. La persistencia ocurre solo después de una mutación programática exitosa. Esta es una transacción compensatoria: los efectos del ciclo de vida pueden ser brevemente visibles, y un rollback fallido se reporta como `AggregateError` en lugar de representarse falsamente como un árbol conservado.

Include lee y valida el contenido candidato desacoplado, aplica los parches a un clon, reconcilia el árbol del Loader y solo entonces confirma el contenido en caché y los datos parseados. `refresh()` rechaza ante su llamador después de un fallo de parseo, validación, aplicación o rollback. La carga inicial sigue siendo de fallo ruidoso; solo un archivo ausente puede usar `initial`. Un resultado YAML/JSON que no es un array es inválido, y tanto el refresh de archivo como la actualización de config de Include vuelven a aplicar los parches sin mutar el parseo en caché.

HMR contiene el rechazo del refresh en vivo. Su método `registerConfig(filename, refresh)` observa una ruta exacta desde el ancestro existente más cercano, serializa y fusiona los refreshes, y devuelve un disposer asíncrono que cierra el watcher y drena el trabajo activo. Tanto los refreshes de ruta exacta como los de archivos de configuración ordinarios usan esa cola. Un fallo se normaliza a `Error`, se registra y se difunde por el evento paralelo `hmr/config-update-failed(filename, error)`; los observadores que rechazan se registran sin detener los refrescos posteriores. Se observan la creación, el cambio y la eliminación.

## Alternativas consideradas

**Contener los fallos dentro de `Include.refresh()`.** Rechazada porque impide que un host de HMR difunda el fallo y aun así permite que la reconciliación del Loader oculte la aplicación parcial. Include es dueño del parseo y la confirmación del candidato; HMR es dueño de la contención y la observación.

**Reiniciar el proceso con cada edición de configuración.** Rechazada porque los efectos de Cordis ya proporcionan un ciclo de vida de plugins reversible, y un error de sintaxis o un plugin opcional fallido no debe descartar sesiones en vivo solo para recuperar la composición previa.

**Prometer un reemplazo atómico invisible.** Rechazada porque los efectos arbitrarios de los plugins no se pueden capturar en instantáneas. La aplicación esperada más la compensación explícita proporciona un resultado final estable sin afirmar que los observadores no pueden ver las transiciones intermedias del ciclo de vida.

## Consecuencias

- Un refresh en vivo fallido rechaza internamente, conserva o restaura el último árbol bueno cuando la compensación tiene éxito y difunde un fallo tipado sin convertirse en un rechazo no manejado.
- Un fallo de rollback es visible y puede dejar una entrada no disponible; el evento y el registro no afirman lo contrario.
- Los fibers que esperan dependencias declaradas siguen siendo entradas pendientes válidas: el asentamiento del ciclo de vida significa que ningún trabajo actual falló, no que toda dependencia exista.
- Los watchers de configuración exactos añaden recursos del sistema de archivos solo para las rutas registradas y los liberan con su fiber HMR propietario.
- El Loader, Include, HMR y el tipado de eventos del core vendidos divergen más de upstream; la divergencia completa se mantiene en el manifest (manifiesto) del vendor.

## Pruebas

`packages/boot/app-boot/tests/config-reload.spec.ts` arranca árboles Loader/Include temporales reales y cubre el rechazo de parseo y forma, importar-antes-de-deshacer, restauración de plugin/configuración, rollback multi-entrada, deshabilitación de ancestros, convergencia de overlays, identidad de opciones, persistencia fallida de actualización directa y movimientos programáticos fallidos. `packages/boot/app-boot/tests/hmr-config.spec.ts` cubre rutas exactas existentes y ausentes, adición/cambio/eliminación, coalescing serializado, drenaje de disposer, normalización de no-`Error`, difusión de fallo y contención de observadores que rechazan. `packages/host/webserver/tests/webserver.spec.ts` demuestra que un fallo de arranque con gate de servicio rechaza la composición del Loader con su diagnóstico de binding; `packages/typert/loader/tests/loader.spec.ts` ejercita la eliminación programática esperada a través de un consumidor real del Loader, y la instantánea de `pty-tools` de ACP (Agent Client Protocol) protege la composición concurrente de reordenar secciones de prompt de igual prioridad.
