# Agent Note: Imágenes Markdown remotas en Web

Status: implemented

[English](2026-07-30-web-remote-markdown-images.md) | Español

## Problema

El Markdown del asistente puede nombrar diagramas y capturas con la sintaxis de imagen estándar, pero el renderizador Web reemplaza cada imagen por texto alternativo en cursiva. Incluso los destinos HTTP(S) absolutos perdían por tanto el comportamiento Markdown ordinario.

## Decisión

`MarkdownText` renderiza los destinos de imagen HTTP(S) absolutos como elementos `<img>` diferidos y responsivos con decodificación asíncrona y `referrerPolicy="no-referrer"`. Las rutas relativas, las rutas locales absolutas, las URLs `file:` y los esquemas no soportados conservan el fallback existente de texto alternativo. El HTML crudo sigue deshabilitado, así que un asistente no puede saltarse el componente de imagen Markdown con un `<img>` escrito a mano.

El componente de imagen reutiliza la política de URL absolutas del renderizador sin añadir un proxy del host, una ruta de archivo local, dependencia de Session, sanitizador o buscador de imágenes. La historia finalizada, la salida en streaming, los parciales interrumpidos y cualquier otro consumidor de `MarkdownText` reciben el mismo comportamiento.

## Alternativas consideradas

**Mantener todas las imágenes como texto alternativo.** Esto preserva la frontera de red más pequeña pero frustra la necesidad del producto de inspeccionar en línea artefactos visuales alojados en la red.

**Proxear las imágenes remotas a través del host.** Un proxy podría ocultar la dirección de red del navegador al origen de la imagen, pero obligaría al host a realizar fetches salientes arbitrarios y requeriría una política separada de redirecciones, DNS, tamaño y contenido. La carga HTTP(S) directa mantiene esa petición visible para los controles del navegador; omitir el referrer limita la divulgación del origen de la conversación.

**Soportar rutas locales en el mismo cambio.** Los orígenes Web no pueden cargar directamente archivos del host. Una implementación segura necesita una frontera de autoridad revisada por separado, así que las rutas relativas, las rutas locales absolutas y las URLs `file:` permanecen deshabilitadas.

**Permitir imágenes `data:`.** Las URLs de datos grandes duplican contenido binario en el texto duradero del transcript. La política de solo HTTP(S) cubre la necesidad actual sin expandir los registros de sesión.

## Consecuencias

Las respuestas del asistente muestran imágenes remotas durante el streaming y el replay sin cambiar los eventos de sesión ni los protocolos del host. Los orígenes remotos siguen observando la petición de imagen, la dirección de red del cliente y cualquier credencial que la política del navegador permita para ese origen. Los destinos locales y no soportados permanecen como texto alternativo inerte.
