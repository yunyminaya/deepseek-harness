# Agent Note: Mantén el almacenamiento del navegador propiedad de jsdom en Vitest

Status: implemented

[English](2026-07-30-vitest-jsdom-webstorage-ownership.md) | Español

## Problema

El rango de Node soportado incluye releases que reservan un `globalThis.localStorage` a nivel de proceso. Node 26 expone esa propiedad como `undefined` sin `--localstorage-file`; Vitest ve la clave reservada y no proyecta sobre ella el objeto `Storage` aislado de jsdom. Las suites de componentes fallan entonces antes de ejercitar el comportamiento de producto, mientras que el carril principal de cobertura de Node 24 permanece verde porque ese runtime no reserva la clave por defecto.

## Decisión

Los workers de Vitest desactivan el Web Storage a nivel de proceso de Node cuando el runtime anuncia la bandera `--webstorage`. La configuración pasa `--no-webstorage` a través del `execArgv` de cada proyecto de test; los runtimes sin esa bandera no reciben ningún argumento. Las suites de entorno Node permanecen por tanto libres de navegador, y los archivos que seleccionan jsdom mediante `@vitest-environment jsdom` reciben el `localStorage` aislado de jsdom.

El agregado de compatibilidad de Node corre un smoke de jsdom dedicado en cada línea de compatibilidad anunciada. Afirma tanto el argumento condicional del worker como el almacenamiento utilizable, de modo que un cambio futuro de Node o Vitest no pueda dejar la suite principal de Node 24 como la única señal.

## Alternativas consideradas

- **Fijar `NODE_OPTIONS=--no-webstorage` en los scripts de paquete o en CI.** Rechazado porque filtra política del test runner a los subprocesos y omite las invocaciones directas de `pnpm exec vitest`.
- **Pasar `--localstorage-file` a Node.** Rechazado porque un store persistente único a nivel de proceso tiene semántica de titularidad y aislamiento distintas del almacenamiento de navegador creado por entorno jsdom.
- **Parchear `globalThis.localStorage` en el código de setup o proteger cada test de componentes.** Rechazado porque el setup dependería de los detalles privados de proyección jsdom de Vitest, mientras que las protecciones por test ocultan un entorno de navegador roto y duplican política entre suites.
- **Fijar los tests a Node 24.** Rechazado porque el engine del paquete anuncia líneas de Node pares más nuevas y la matriz de compatibilidad existe para exponer sus cambios de runtime.

## Consecuencias

El mismo comando `pnpm test` funciona en releases de Node con y sin Web Storage integrado. Los test workers deliberadamente no pueden ejercitar el Web Storage a nivel de proceso de Node; una necesidad futura de producto de esa API requiere una configuración de test explícita separada en lugar de debilitar el aislamiento de jsdom. El carril de compatibilidad añade un proceso de Vitest enfocado en lugar de duplicar el inventario unitario completo en cada versión de Node.
