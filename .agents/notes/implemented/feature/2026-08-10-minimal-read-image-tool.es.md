# Agent Note: Una herramienta read_image mínima sobre los seams existentes

Status: implemented

[English](2026-08-10-minimal-read-image-tool.md) | Español

## Problema

El trabajo de adjuntos multimodales dio a las subidas de usuario una ruta durable completa, pero el propio modelo no tenía forma de inspeccionar una imagen en disco. `read` rechaza el contenido binario por contrato, así que un agente al que se le preguntaba por una captura de pantalla o un gráfico renderizado o fallaba o usaba una solución alternativa con pérdida. Un intento independiente en el PR #598 combinaba la herramienta con el scoping de rutas a nivel de loop, visibilidad de schema por ruta y nuevos conceptos de log de sesión. Esas funcionalidades no eran necesarias para publicar un resultado de herramienta de imagen registrado.

## Decisión

Ambas operaciones de lectura de imagen viven en `dsh-tool-fs` y publican resultados ordinarios de herramienta registrados sobre los puntos de extensión existentes.

- **`read_image` lee una ruta del filesystem.** La extensión selecciona el tipo de medio declarado PNG/JPEG/WebP/GIF; la validación de magic bytes y píxeles del store de adjuntos sigue siendo autoritativa. Los bytes viajan `ctx.fs.stat` → `ctx.fs.readBytes` acotado → `ctx.attachments.saveImage` → `fs/observed`. El resultado de la herramienta contiene metadatos y un `ImageBlock`.
- **`FileSystem.readBytes(target, signal, maxBytes)`** es una primitiva de provider obligatoria nueva: el límite de bytes vive en el seam para que ningún backend pueda almacenar en búfer un archivo sin límite, con el cortocircuito de tamaño de stat y una guardia de stream de un byte sobre el tope contra el crecimiento posterior al stat (`FS_TOO_LARGE`).
- **El registro es condicional a la composición; la ejecución está restringida por ruta.** Las herramientas se registran solo bajo `ctx.inject(['attachments'], …)`. Antes del I/O, el gate estricto resuelve la ruta llamante a través de `ctx.llm.resolveModelInfo` y exige `image` en `inputModalities`; la capacidad desconocida se niega. Una ruta solo-texto puede consumir aun así imágenes durables previas porque el runtime LLM compartido las proyecta a placeholders en el ensamblaje de la solicitud.
- **Code Mode reenvía la imagen fuera de banda**: un despacho anidado devuelve el valor canónico (local a la ejecución, sin bloque de imagen) y difiere un mensaje de contexto de rol `user` que porta el sobre y la imagen, así que la imagen llega igualmente a la siguiente solicitud.
- **Los modelos de llm-replay pueden declarar `inputModalities`**, lo que permite que las instantáneas ACP sin clave cubran el resultado con capacidad de imagen y la negativa solo-texto.

## Alternativas consideradas

- **El diseño con scoping de rutas del PR #598** usaba un punto de extensión listo para solicitud, visibilidad de schema por ruta, proyección reversible y tres conceptos durables. La proyección de solicitudes LLM compartida ahora maneja las rutas solo-texto sin meter el registro de herramientas ni los formatos de sesión en el agent-loop.
- **`agent.inject()` en lugar del resultado de herramienta con imagen** — enruta la imagen alrededor del resultado de la herramienta como un mensaje de usuario inyectado separado. Rechazado: la imagen ES el resultado de la herramienta; dividirlos añade un segundo mensaje registrado sin ganancia, y la ruta de resultado de herramienta ya funciona de punta a punta.
- **Detección de magic bytes en lugar de declaración de extensión** — el sniffing duplica la detección que el store de adjuntos ya posee (respaldada por sharp, autoritativa). La extensión es solo una *declaración*; un desajuste falla de forma cerrada con un remedio de renombrado en lugar de aceptarse en silencio, lo que también mantiene honesto el mapa mental del modelo (nombre de archivo ↔ contenido).
- **Registrar incondicionalmente y fallar ante un store ausente** — rechazado; un despliegue sin store de adjuntos no puede satisfacer nunca la herramienta, así que su schema sería una mentira permanente. El gate de ruta, en cambio, es estado por llamada y vive correctamente en el límite de ejecución.

## Consecuencias

- Las herramientas se niegan a ejecutarse en una ruta solo-texto, mientras que las imágenes existentes en el historial de sesión están representadas por placeholders locales a la solicitud.
- Los resultados de imagen repetidos acumulan coste de solicitud hasta que la proyección de solicitud o la compactación los elimina; el direccionamiento por contenido deduplica los bytes durables.
- La tarjeta de resultado de herramienta renderiza la referencia durable, no los píxeles; la vista previa inline se difiere a los paquetes de UI.
