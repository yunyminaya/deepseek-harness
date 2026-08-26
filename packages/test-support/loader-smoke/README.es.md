# `@deepseek-ai/dsh-loader-smoke`

[English](README.md) | Español

Harness de subprocesos compartido para tests que arrancan una app y un `cordis.yml` a través del Loader de Cordis. `resolveExampleLaunch` selecciona el modo `src` local (tsx y rutas tsconfig raíz) o el modo `lib` de CI (Node plano y exports de paquete) a partir de un modo explícito o `DSH_EXAMPLE_MODE`.

`runLoaderSmoke` acepta rutas de bin y config, argumentos completos de bin opcionales, sobrescrituras de entorno, stdin, setup previo a la ejecución e inspección previa a la limpieza. Es dueño del cwd aislado, los homes DSH, los diagnósticos, el plazo, la terminación, el EOF y la limpieza; devuelve ambos streams tras una salida cero y rechaza con ambos streams ante un fallo.

`runFixtureTurn` conduce una tarea a través de exactamente un agent raíz configurado, reenvía los eventos canónicos después de que esa tarea llega a la bandeja de entrada duradera, vacía la sesión y devuelve el texto final del assistant más el uso acumulado. Los drivers locales al ejemplo conservan la propiedad de la configuración, el renderizado y la aserción.

Esto es infraestructura de pruebas de nivel soporte, no API de producto.

## Model Experience

Ninguna, ya que el harness de pruebas envía solo la tarea de usuario ordinaria del test consumidor y delega la composición de prompt y herramientas al árbol cargado.

#### Efecto de caché KV

Ninguno más allá del árbol cargado; el helper ni cambia el prefijo de petición ni conserva estado entre ejecuciones.

## Limitaciones conocidas y trabajo diferido

- **El modo compilado requiere un build previo** — la config también debe resolver cada paquete nombrado hacia arriba a través de `examples/node_modules`.
- **El stdout y stderr capturados están acotados solo por el `maxBuffer` por defecto de execa de 100 MB** — un hijo desbocado se termina en ese techo en lugar de en un presupuesto elegido por el smoke.
- **El tiempo de espera mata solo al hijo directo** — un árbol de procesos generado por un fixture defectuoso puede sobrevivir al smoke y necesita limpieza externa.
