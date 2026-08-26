# Agent Note: Los cambios de la GUI web cierran el bucle sobre la URL existente

Status: implemented

[English](2026-07-28-web-gui-feedback-loop.md) | [中文](2026-07-28-web-gui-feedback-loop.zh.md) | Español

## Problema

El agent web no podía identificar ni la GUI que alojaba su sesión ni la URL que el usuario estaba viendo. La [decisión de contexto de runtime](2026-07-28-web-agent-runtime-context.es.md) aporta el primer hecho, pero una edición de la GUI seguía sin tener un objetivo de aceptación ejecutable: las ediciones de fuente, las construcciones de artefactos, un proceso escuchando y la página existente del usuario eran observaciones sin relación. Los privilegios del repositorio hacían ver válido a un sustituto equivocado porque `apps/web/package.json` exponía `vite` como su script `dev` y el Vite pelado devolvía HTTP 200 aunque no podía inyectar `window.__DSH_BOOT__`.

El [post-mortem del incidente](../../../../docs/postmortem/0003-web-agent-gui-feedback-loop.es.md) es dueño de la cronología del registro de eventos y del porqué las comprobaciones originales aceptaron la página, el proceso y el puerto equivocados.

## Decisión

La composición `dsh web` ordinaria monta el plugin `web-runtime` del bundle web, que publica una URL canónica de loopback tanto como orientación visible para el modelo como como hecho gestionado del shell. La sección de prompt `app:web-surface` dice que las referencias sin calificar identifican esta GUI y nombra la URL; `DSH_WEB_URL` lleva el mismo hecho a cada llamada bash en primer plano o gestionada en segundo plano. La sección preserva la frontera de sin-DOM-implícito, ruta o captura de pantalla, y no afirma que un alias de LAN equivalga a la dirección literal del navegador. Un perfil de prompt completo pone el `surfaceContext` de la fila en false y no recibe ni la sección de prompt ni la variable gestionada; el lanzador web usa el mismo ajuste para suprimir su sección de prompt de checkout de fuente.

El prompt hace que el agent, y no el usuario, sea dueño del contrato oculto de arranque. El receptor HMR de plugins de cliente siempre está montado, pero la recarga automática de plugins de cliente exige además un watcher `pnpm run dev:web` del mismo checkout, cosa que el agent verifica antes de prometer actualizaciones sin refresco. Los cambios del shell y de otros paquetes simples siguen exigiendo reconstruir los artefactos afectados y refrescar la URL existente. El agent no lanza una GUI de reemplazo salvo que se le pida.

El script de desarrollo de `apps/web` y la configuración de Vite rechazan el modo serve antes de abrir un puerto. Sus diagnósticos identifican `apps/web` como un shell de solo construcción, explican que solo `dsh web` inyecta `window.__DSH_BOOT__` y nombran las rutas de entrada de producción y de HMR. El modo build de Vite permanece sin cambios.

No se exige reiniciar ni reemplazar el servidor meramente porque cambiaron artefactos estáticos. El host lee `index.html` y los recursos estáticos en cada solicitud, mientras que los bundles de cliente también se sirven de sus archivos actuales con `no-cache`; un refresco de la URL existente es por tanto la vía de aceptación después de reconstruir los bundles de shell y de plugin pertinentes. Arrancar un servidor aparte solo demuestra que un servidor aparte funciona. Si el usuario pide explícitamente otro servidor de larga duración, el contrato existente de tareas en segundo plano gestionadas es dueño de su ciclo de vida y de sus avisos de finalización; el `&` del shell no es un ciclo de vida alternativo.

## Verificación

El escenario sin clave de navegador de ida y vuelta en frío arranca la composición web publicada, conduce una sesión real reproducida, captura el prefijo del system-prompt que porta la URL e invoca la herramienta bash ensamblada para probar que `$DSH_WEB_URL` coincide con el runtime realmente enlazado. El smoke real de CLI lanza `dsh web` y captura la solicitud al provider, fijando el contrato de desarrollo completo de dos comandos. La prueba del watcher `dev:web` reconstruye un bundle de cliente aislado tras un cambio de fuente; el escenario HMR de navegador lanza `dsh web`, cambia un bundle inicial de roster y observa el nuevo DOM bajo la misma identidad de página. Una prueba real de subproceso Vite exige que el modo serve termine de forma natural con la corrección de host completo e instrumenta `Server.listen()` para probar que nunca se llamó. La prueba real de servidor web con Loader real reescribe un recurso estático después de que el proceso se enlace y prueba que el mismo puerto devuelve los nuevos bytes. Estas aserciones inspeccionan estado del prompt, salida de proceso, salida del shell, identidad del DOM y bytes HTTP en lugar de una declaración de éxito del agent.

## Alternativas consideradas

**Extender solo el system prompt.** Rechazado porque dejaría el objetivo inaccesible para las herramientas, preservaría la engañosa vía del Vite pelado y no demostraría cómo un proceso existente observa los artefactos reconstruidos.

**Eliminar el script de desarrollo de `apps/web` sin blindar Vite.** Rechazado porque `npx vite`, el comando exacto del incidente, esquiva los scripts del paquete. El modo serve en sí debe fallar.

**Reiniciar o reemplazar automáticamente el proceso web actual tras cada edición.** Rechazado porque el servidor estático ya lee los artefactos actuales por solicitud, un reinicio interrumpiría la sesión que pidió la edición, y la recarga de plugins de cliente es propiedad de la cadena HMR siempre montada más el watcher `pnpm run dev:web`.

**Enviar DOM, ruta o capturas de pantalla con cada solicitud.** Aplazado a un diseño aparte de entrada registrada. La identidad estable de la URL cierra este bucle de retroalimentación sin afirmar un estado del navegador que el host no recibe.

## Consecuencias

Los prompts web ordinarios ganan un párrafo dinámico de URL, así que la reutilización del prefijo al provider ahora varía según el puerto enlazado. Sus procesos bash ganan una variable de entorno gestionada no secreta más. El Vite pelado ya no sirve como sandbox visual de solo shell; los desarrolladores usan el host completo o el modo build en su lugar. A cambio, el trabajo de GUI tiene un objetivo mecánicamente observable, el agent puede enseñarle al usuario el comportamiento de actualización exacto del proceso que de hecho sirve su sesión, y la vía de arranque no soportada falla antes de una pantalla en blanco. El contrato de URL guía al agent lejos de los puertos de reemplazo; no prohíbe que comandos de shell arbitrarios inicien uno. Los perfiles que deshabilitan `surfaceContext` también renuncian a esta guía de bucle de retroalimentación y al contexto del shell.
