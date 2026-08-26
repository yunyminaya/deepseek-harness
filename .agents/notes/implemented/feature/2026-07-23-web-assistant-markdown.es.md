# Agent Note: Markdown seguro del asistente en la conversación web

Status: implemented

[English](2026-07-23-web-assistant-markdown.md) | Español

## Problema

La conversación web conserva el Markdown fuente del asistente a través de los eventos de sesión, la reproducción del historial y la acumulación en streaming, pero su primitiva de texto terminal renderiza esa fuente literalmente. Cambiar la primitiva compartida formatearía también los mensajes de usuario y de steering, mientras que parsear en el runtime mezclaría estado de presentación en la proyección de sesión sin React.

## Decisión

`@deepseek-ai/dsh-client-ui-primitives` exporta `MarkdownText` como el renderizador de texto de asistente no confiable, y `ui-conversation` lo selecciona solo para los bloques `text` de asistente. El historial finalizado, la cola de streaming y los parciales interrumpidos ya comparten `AssistantMarkdown`, por lo que reciben el mismo renderizador sin cambiar eventos ni instantáneas. Los mensajes de usuario y de steering mantienen `MessageText` y siguen siendo literales.

`MarkdownText` parsea con `mdast-util-from-markdown` más las extensiones micromark GFM y renderiza el árbol mdast con el renderizador propio del paquete, parseando de forma incremental mientras un turno transmite (la [nota del renderizador AST incremental](../architecture/2026-08-06-web-markdown-incremental-ast-renderer.es.md) es la dueña de ese mecanismo y de su contrato de paridad DOM). Cubre los bloques CommonMark más las tablas GFM, las listas de tareas, el tachado y los autolinks sin parsear HTML bruto. Una extensión de atención micromark reutiliza el resolver CommonMark mientras permite que las corridas de al menos dos asteriscos cierren después de puntuación Unicode cuando van seguidas inmediatamente de texto CJK. Esta excepción cubre el énfasis fuerte terminado en puntuación en prosa CJK sin espacios durante el streaming y después del asentamiento; el énfasis de un solo asterisco, la adyacencia no CJK, la fuente escapada, el código y las matemáticas conservan el parseo upstream. El código delimitado se enruta a través del `CodeBlock` compartido, que resalta las gramáticas registradas con el singleton shiki del cliente (tokens `--shiki-*`) y vuelve a la monoespaciada plana en caso contrario. Mientras un turno transmite, los fences permanecen en la rama plana para que los fences en crecimiento no se retokenicen en cada fragmento.

El espaciado visual, las tablas, los enlaces, los blockquotes, el código en línea y el marco del bloque de código siguen a `@deepseek/md` de deepsuite (`markdown.css` / `code-block.css`) y a los mismos tokens `--dsw-alias-markdown-*`, `--dsw-font-markdown-*`, `--dsw-alias-border-l*` y `--dsw-alias-label-*`. Los enlaces usan `--dsw-alias-state-business-primary` (la hoja de deepsuite usa `--dsw-alias-brand-text`, que es azul solo bajo newDesign; design-platform mantiene brand-text casi negro y no se reajusta aquí). Cuando un token de código en línea consiste enteramente en una URL absoluta HTTP(S), su marco de código contiene el mismo ancla externa segura enfocable por teclado que un enlace ordinario; el puerto, la ruta y el texto de consulta permanecen sin cambios, mientras que los comandos, las URLs parciales, otros esquemas y el código delimitado siguen inertes. `CodeBlock` incluye una etiqueta de lenguaje y un control de copiado (`复制` / `复制成功`). El texto finalizado renderiza KaTeX a través de las extensiones matemáticas de la gramática asentada; `mathCompatibility` mapea `\(...\)`, `\[...\]` y los `$$...$$` de nivel de bloque en la misma línea a los mismos nodos AST matemáticos estándar. Esta es una capa estrecha de compatibilidad de parser, no una reescritura con regex ni una reparación de salida de modelo malformada. El streaming sigue siendo literal hasta la finalización para que las fórmulas incompletas no muestren errores intermitentes. Las píldoras de citas, las anclas de encabezados, la variante de markdown thinking-small y los marcadores de tarea □/☑ personalizados quedan fuera de alcance; las listas de tareas GFM conservan sus checkboxes nativos.

La dependencia es explícita en `ui-primitives`; como esa biblioteca pura es sembrada por el shell web, el parser y el resaltador forman parte del bundle inicial del navegador.

## Política de salida no confiable

Los destinos de enlace escritos por el asistente están restringidos a URLs absolutas HTTP, HTTPS y mailto. Los enlaces HTTP(S) se abren en una pestaña nueva con `rel="noopener noreferrer"`; los destinos relativos y otros protocolos se renderizan como texto no navegable. Las imágenes Markdown siguen la [política de imágenes remotas](2026-07-30-web-remote-markdown-images.es.md) aparte. El HTML bruto sigue siendo texto fuente inerte porque ningún parser HTML entra en el pipeline. La salida de Shiki es un árbol de spans estático generado desde el texto del fence (sin scripts ni HTML de usuario).

El código delimitado y las tablas GFM son dueños del desbordamiento horizontal para que el contenido largo no pueda ensanchar la columna de conversación.

## Alternativas consideradas

**Promover las dependencias de desarrollo mdast y micromark existentes y mantener un walker React personalizado.** Esto evita una familia de parser nueva, pero hace que el producto sea dueño de cada mapeo de nodos, extensión GFM y rama de renderizado sensible a la seguridad. El renderizador React dedicado mantiene ese recorrido upstream conservando una ruta de AST a React. *Luego revertido por nueva evidencia: el parseo incremental en streaming necesita entrada a nivel de AST que el wrapper solo de cadenas no puede proporcionar; la [nota del renderizador AST incremental](../architecture/2026-08-06-web-markdown-incremental-ast-renderer.es.md) es la dueña de esa decisión.*

**Sustituir `MessageText` por renderizado Markdown.** Esto formatea los prompts de usuario y el steering como efecto secundario. Esas entradas escritas siguen siendo literales hasta que el producto elija ese comportamiento explícitamente.

**Parsear Markdown en las instantáneas de sesión.** Esto haría que los nodos React o los ASTs de presentación fueran estado de runtime duradero y reintroduciría un límite de modo final-versus-streaming. El parseo se queda en cambio en la hoja de presentación.

**Habilitar HTML bruto con sanitización.** El HTML bruto no tiene ninguna necesidad de producto actual y ampliaría el límite de contenido ejecutable, por lo que permanece deshabilitado en lugar de añadir una dependencia de sanitizador. Las imágenes remotas se rigen por la [política de imágenes](2026-07-30-web-remote-markdown-images.es.md) posterior.

**Portar `highlight.css` de Prism de deepsuite y el pipeline mdast.** La paridad de apariencia es propiedad de CSS Modules y de los tokens `--dsw-*` compartidos; el resaltado se queda en la allowlist shiki existente para que el cliente no tome un segundo resaltador ni un contrato de clases Prism.

**Preprocesar la fuente Markdown o reparar los nodos de texto tras el parseo para los límites de puntuación CJK.** Una reescritura de fuente debe reproducir las reglas de escape, código, matemáticas y delimitadores antes de que el parser sea dueño de esas distinciones, mientras que una reparación de nodos de texto ya ha perdido parte de la intención de la fuente y no puede componerse con los nodos en línea parseados. Extender la atención en el límite del tokenizer conserva el resolver upstream y limita la divergencia a la elegibilidad de delimitadores.

**Exigir al modelo que emita enlaces estándar y dejar el código en línea con forma de URL inerte.** La guía de salida no puede uniformar las respuestas de modelos persistidos o de terceros, y el código en línea es una forma común de distinguir un endpoint literal. Reconocer solo un valor absoluto HTTP(S) completo en el límite del código en línea renderizado conserva la semántica del código mientras aplica la política de enlaces no confiables existente.

## Consecuencias

Las respuestas del asistente renderizan Markdown semántico de forma consistente durante el streaming y la reproducción, mientras que las tarjetas de herramientas, las filas de razonamiento, las interacciones, las burbujas de usuario y el protocolo del host permanecen sin cambios. El streaming reparsea solo la cola inestable tras cada actualización acumulada; el Markdown incompleto puede cambiar temporalmente la estructura de la cola, pero la cola aislada acota la invalidación de React y el evento final no cambia de renderizador. El código en línea con forma de URL se vuelve navegable sin cambiar su literal visible, mientras que los esquemas inseguros y el código mixto siguen siendo no interactivos. Los fences de código comparten un único marco y ruta de copiado con las superficies de herramientas y detalles. El shell web inicial incluye el parser Markdown, el runtime GFM, KaTeX y la allowlist shiki; las superficies de citas, anclas y thinking-small siguen diferidas.
