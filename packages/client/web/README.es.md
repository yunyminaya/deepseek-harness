# @deepseek-ai/dsh-client-web

[English](README.md) | Español

Kernel de arranque web: `new AppWebEntry(el, seams?).run()` monta el cliente en dos etapas. La etapa de módulos llama a `window.__ModuleLoader__.create()`, instalado por el Host, con `window.__DSH_BOOT__`, los módulos estáticos del shell y cualquier override de transporte de prueba; la fachada devuelve el sistema de módulos construido y el manifest parseado tras adoptar los registros precargados por el parser. Este paquete precarga entonces el nivel `immediately`. La etapa de plugins monta el Loader de Cordis vendido, inyecta ese sistema de módulos a través de la interfaz `internal` del Loader, crea cada entrada del grafo de forma uniforme y espera a que cada fiber se vuelva ACTIVE. Después entrega el DOM de arranque marcado a la operación `ctx.uiRenderer.mount(el)` del renderizador de UI dinámico; el renderizador hidrata ese DOM antes de cambiar a la UI completa. El Host es dueño del grafo, de las precargas del parser y de la fachada; AppWebEntry no conoce el id del paquete de bootstrap ni parsea el formato de cable.

La página de arranque usa DOM simple y CSS local, de modo que los fallos del bundle de cliente y de la activación de plugins permanecen visibles. Sus fuentes y colores de respaldo coinciden con los tokens de tema que llegan durante la carga. Las actualizaciones de fiber conservan un nodo de spinner y hacen crecer su arco CSS a medida que las entradas se vuelven activas por primera vez; la hidratación preserva ese nodo y su fase de animación hasta el commit de la aplicación. El montaje de React, el renderizado de slots, el ensamblado de la aplicación y la proyección del título del navegador viven en [`ui-renderer`](../ui-renderer/README.es.md). El bundle de módulos cachea sus propios exports materializados y proporciona el sistema capturado por cierre cuando su entrada ordinaria del grafo se activa; la espera de servicios de Cordis hace que el orden de creación de las filas del grafo sea independiente de esa activación.

`PLATFORM_MODULES` (src/platform.ts) es la única fuente de verdad para los módulos compartidos sembrados por el shell. Junto con `PRELOADED_CLIENT_EXTERNALS`, define la línea base externa implícita de cada bundle dinámico; `dsh.client.external` añade solo las solicitudes exactas fuera de la línea base.

El parámetro de override opcional `seams` reenvía el override de transporte `loadBundle` del sistema de módulos (`BootSeams`) para entornos donde la ejecución de `<script>` externos no puede alcanzar el contexto de la página; los llamadores ordinarios de navegador lo omiten. Un transporte de página preinyectado es el valor por defecto por delante de él: cuando `globalThis.__DSH_TRANSPORT__` (los `ClientTransportHooks` del paquete de conexión) lleva `loadBundle`, la etapa de módulos lo adopta como transporte de bundles y se salta el prefetch HTTP del nivel inmediato — los `seams` explícitos siguen ganando.

## Experiencia del modelo

Ninguna, ya que el shell de entrada arranca el árbol de plugins del navegador; nada de aquí llega a una solicitud de modelo.

#### Efecto en la caché KV

Ninguno; este paquete ni ensambla ni envía una solicitud de provider.

## Limitaciones conocidas y trabajo diferido

- **La aplicación espera al roster completo** — una entrada fallida mantiene visible la página de arranque sin framework con un informe por entrada; no se soporta la disponibilidad parcial de la UI.
