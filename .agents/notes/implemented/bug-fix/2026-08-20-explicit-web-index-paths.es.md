# Agent Note: Rutas de índice Web explícitas y fallos 404 de estáticos

Status: implemented

[English](2026-08-20-explicit-web-index-paths.md) | Español

## Problema

Un fallback SPA incondicional hace que toda petición GET o HEAD sin coincidencia parezca exitosa. Un enlace ordinario roto y un JavaScript, hoja de estilos, source map o manifest ausentes reciben entonces la carcasa HTML con estado 200, de modo que navegadores, cachés y monitorización no pueden distinguir una entrada de página válida de un recurso ausente.

## Decisión

`dsh-host-frontend-static` renderiza `index.html` solo cuando el destino normalizado es la raíz de dist o la ruta de índice configurada. El cliente Web actual no tiene rutas de pathname de History API; las query strings no cambian la coincidencia de pathname, y los fragmentos de URL nunca llegan al servidor. Los archivos existentes se sirven con normalidad, mientras que las lecturas `ENOENT`, `EISDIR` y `ENOTDIR` producen una respuesta 404 vacía sin tipo de contenido. Los demás fallos de filesystem se relanzan al manejo de fallos de petición del servidor web en lugar de etiquetarse como ausencia.

GET y HEAD usan el mismo estado y tipo de contenido para entradas de índice, archivos y fallos. Las rutas nombradas siguen coincidiendo antes que el fallback, el traversal fuera de la raíz de dist sigue siendo 403, y las peticiones no-GET/HEAD que alcanzan el fallback siguen siendo 405.

## Alternativas consideradas

**Inferir las rutas de página de la ausencia de extensión de archivo.** Una extensión de archivo no declara una ruta de cliente: esto seguiría convirtiendo rutas ordinarias desconocidas en páginas exitosas, rechazaría cualquier ruta de cliente futura con puntos y manejaría mal los estáticos sin extensión cuando estén ausentes.

**Usar una cabecera de petición `Accept: text/html` como regla del fallback.** La cabecera expresa preferencia de representación, no si el pathname es una ruta de cliente declarada. Los fetches de navegadores, bots y monitores pueden solicitar HTML para una ruta inválida, así que el mismo comportamiento de falso éxito permanece.

**Añadir ahora una allowlist de pathnames configurable.** Ninguna ruta de cliente actual consume esa configuración. Un router de History API futuro puede añadir una regla de servidor explícita o un campo de configuración junto con la ruta que lo exija, sin preservar hoy una opción pública especulativa.

## Consecuencias

Los enlaces rotos y los assets ausentes tienen un estado HTTP distinto que cachés y monitorización pueden observar, y un cargador de assets no puede ejecutar la carcasa HTML como JavaScript. Una ruta de cliente futura basada en pathname devuelve 404 hasta que su regla de entrada de servidor y su cobertura de composición real aterricen en el mismo cambio. La prueba real de Loader de frontend-static fija la paridad GET/HEAD para entradas de índice, assets existentes, fallos ordinarios y fallos de recursos; también cubre rutas con aspecto de API, traversal, objetivos malformados, métodos no soportados y la disposición del fallback.
