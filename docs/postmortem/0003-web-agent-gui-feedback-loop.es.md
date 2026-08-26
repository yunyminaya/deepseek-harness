# Post-mortem 0003: el Web agent validó un servidor de reemplazo en lugar de su GUI actual

[English](0003-web-agent-gui-feedback-loop.md) | Español

Estado: resuelto

## Resumen ejecutivo

Un Web agent (agente) cambió el código fuente de la GUI pero no sabía qué URL y qué proceso alojaban su sesión. Delegó la aceptación en el usuario, luego trató un HTTP 200 desnudo de Vite como éxito pese a una pantalla blanca por la ausencia de `window.__DSH_BOOT__`, y finalmente validó un servidor de reemplazo `dsh web` en otro puerto mientras la página original ya había recogido los artefactos recompilados. La corrección hace que la URL actual y el modo de ejecución sean visibles para el modelo y consultables desde el shell, rechaza Vite en solitario antes de escuchar, y verifica la actualización de producción y el HMR de desarrollo contra estado externo.

## Resumen

La sesión se ejecutaba dentro de la Web GUI de DeepSeek Harness en el puerto 3081 mientras su Workspace seleccionado era un directorio `test/` vacío. La solicitud del modelo no nombraba ni la GUI ni su checkout de código fuente, URL, proceso o modo de actualización. Las posibilidades del repositorio exponían `apps/web` con un script de desarrollo de Vite, mientras que la composición completa del navegador vivía detrás de `dsh web`.

Las acciones resultantes eran individualmente plausibles, pero no compartían un único objetivo de aceptación. Una edición del código fuente, una compilación exitosa, un HTTP 200, un manifest de arranque inyectado y la página existente del usuario se trataron como hechos intercambiables.

La fuente de evidencia es el registro de eventos persistido de `session-3eb796c2-5159-4686-affe-df8719f6f987`, cuya cabecera registra el cwd `/Users/tn.shen/Documents/deepseek-harness-gui-master/test`. Su cabecera de solicitud inicial es la secuencia 6; la transferencia al usuario, el lanzamiento de Vite en solitario, el lanzamiento del host de reemplazo, la sonda del manifest de arranque y la primera sonda del proceso 3081 son las secuencias 30939, 31865, 34309, 34441 y 34681, respectivamente. La cronología siguiente sigue esos eventos en lugar de reconstruir la intención a partir del informe posterior.

## Impacto

El usuario tuvo que detectar tres errores consecutivos: la aceptación se le devolvió a él; la vista previa propuesta era una página en blanco; y la URL reportada como exitosa no era la página que estaba usando. Además, un servidor de reemplazo sin gestionar sobrevivió al turno hasta que el usuario lo cuestionó.

Ningún cambio de esta investigación reinició ni modificó los servicios de prueba de solo lectura 3081 y 3082.

## Cronología

- En el turno 2, tras editar el tema, el mensaje de la secuencia 30939 del agent le dijo al usuario que ejecutara `pnpm run demo:tui` o abriera una aplicación Web sin especificar. No ejecutó ninguna aceptación Web ensamblada.
- En el turno 3, el agent leyó `apps/web/package.json`, lanzó Vite en solitario en el puerto 5173 en la secuencia 31865, observó HTTP 200 y declaró éxito. El navegador, en cambio, lanzó `client-modules: window.__DSH_BOOT__ is missing or not an object` y renderizó una página blanca.
- En el turno 4, el agent encontró la ruta completa de `dsh web`, recompiló el shell, lanzó un proceso sin gestionar en el puerto 3334 en la secuencia 34309 y solo comprobó que este reemplazo devolviera 200 con un manifest de arranque en la secuencia 34441. Nunca sondó el puerto 3081.
- En el turno 5, el usuario informó en la secuencia 34556 de que 3081 ya mostraba el nuevo tema. Solo entonces, en la secuencia 34681, el agent inspeccionó el proceso existente y eliminó el servidor redundante.

## Causa raíz

El ensamblaje Web no tenía identidad visible para el modelo de la GUI actual, de la URL canónica ni del modo de ejecución. El cwd de la sesión identificaba correctamente el Workspace seleccionado por el usuario, pero el modelo trató ese directorio de proyecto como el directorio de la aplicación. Ningún registro duradero relacionaba el checkout del código fuente de la GUI, los artefactos compilados, el proceso de servicio, el origen objetivo y la aceptación del navegador.

La ruta de arranque equivocada parecía legítima porque Vite en solitario devolvía HTTP 200. `window.__DSH_BOOT__` solo lo inyecta el host completo, así que la disponibilidad del transporte no implicaba la disponibilidad de la aplicación. La primera prueba de regresión repitió este error de otra forma: un tiempo de espera agotado mató a Vite y satisfizo una aserción de salida no cero. La reproducción en vivo expuso ese falso positivo.

La semántica de procesos en segundo plano también se eludió con `&` del shell, así que la identidad del job, los avisos de finalización, la recolección y la limpieza no se aplicaron. Por tanto, verificar el puerto 3334 solo demostró que un segundo servicio funcionaba.

## Protecciones añadidas

- El lanzador Web publica la URL canónica de loopback y el modo real de producción/desarrollo en la sección de prompt registrada `app:web-surface` y en el entorno gestionado `$DSH_WEB_URL`/`$DSH_WEB_MODE`.
- La guía de producción exige recompilar los artefactos y verificar la URL existente tras la actualización. La guía de desarrollo explica que `dsh web --dev` monta solo el receptor de HMR; `pnpm run dev:web` en el mismo checkout también debe recompilar los bundles de client-plugin, mientras que los cambios en shell y en paquetes simples siguen requiriendo actualización.
- El modo de servicio Vite independiente de `apps/web` se rechaza durante la configuración. Su prueba de subproceso demuestra la salida natural e instrumenta `Server.listen()` para que un bind transitorio no pase desapercibido.
- Pruebas de ruta real en capas cubren la solicitud CLI, los prompts exactos de producción/desarrollo, los hechos de ejecución del shell, el reemplazo estático en el mismo puerto, la recompilación del observador de código fuente, el sondeo de stat del host y el HMR del navegador con una identidad de página sin cambios.
- La evidencia del PR conserva capturas de pantalla de la sesión original 3081 y una ejecución de GUI antes/después con un modelo real; las observaciones externas de navegador, HTTP, proceso y registro de sesión soportan la aceptación.

## Lecciones

- El agent debe conocer los requisitos previos ocultos de ejecución antes de poder guiar al usuario; el modo de arranque es contexto de la aplicación, no conocimiento tribal.
- La disponibilidad HTTP, el éxito de compilación y un manifest de arranque son hechos distintos. La aceptación nombra el origen exacto y observa externamente el cambio solicitado allí.
- Un servicio de reemplazo no puede demostrar que una página existente cambió. Los procesos de larga duración usan ciclos de vida de tarea gestionados cuando de verdad se solicitan.
- Una prueba de regresión debe poder fallar por el mecanismo reportado. El tiempo de espera del proceso no equivale a fallar rápido, y la disponibilidad del puerto tras la salida no demuestra que el puerto nunca se enlazó.
