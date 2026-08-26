# @deepseek-ai/dsh-client-ui-attachment

[English](README.md) | [中文](README.zh.md) | Español

Plugin dinámico de presentación de adjuntos para la UI de conversación. Espera las declaraciones `conversation.input.attachments` y `conversation.message.images` del paquete de conversación mediante `ctx.slots.inject`, y luego registra el carril de imágenes del borrador del composer, el destino de soltar documentos, la galería de imágenes del historial de chat y el lightbox de la imagen original. El dueño del slot de conversación suministra los datos de adjuntos, la carga de imágenes, los callbacks y su traductor de namespace; los componentes de presentación siguen siendo props puras y no se exportan desde la entrada del paquete.

## Carril de adjuntos

`AttachmentRail` renderiza las imágenes de borrador pendientes como miniaturas fijas de 64px (radio de 16px) en una sola fila de desplazamiento horizontal cuya barra de desplazamiento permanece oculta. El desbordamiento lo anuncian en su lugar flechas circulares en los bordes: cada una pagina un viewport (menos una tarjeta de contexto, con un mínimo de 200px) con desplazamiento suave (instantáneo bajo `prefers-reduced-motion: reduce`), y la visibilidad de las flechas se recalcula desde la geometría de desplazamiento al hacer scroll, al cambiar el número de elementos y al cambiar el tamaño del carril (un ResizeObserver sobre el elemento del carril, de modo que cuentan los redimensionados de la barra lateral y de los paneles, no solo los de la ventana). El carril se desplaza solo en horizontal: un listener no pasivo consume cada tick de rueda con componente vertical — nada desplaza la conversación detrás del composer — convirtiendo una rueda puramente vertical en un paso horizontal (deltas LINE/PAGE normalizados a píxeles, recorrido por tick limitado a 60px) y conservando la intención horizontal de un arrastre diagonal, mientras que los arrastres puramente horizontales siguen siendo nativos. Un elemento recién añadido se revela al final del carril; la eliminación conserva la posición de desplazamiento, y un carril que se monta sobre un borrador ya poblado conserva su posición inicial. Cada miniatura abre su original a través de `onOpen` con un solo clic, y su control de eliminación está en la esquina superior derecha de la tarjeta, oculto hasta que la tarjeta recibe hover o el control recibe el foco del teclado; las superficies de puntero grueso (táctiles) lo muestran permanentemente porque no tienen hover. El dueño decide el montaje y renderiza el carril solo mientras existen elementos.

## Imágenes de mensaje y el lightbox

`MessageImage` renderiza una imagen duradera del historial, cargando una URL autorizada para la sesión a través del `ImageLoader` del dueño; una carga fallida renderiza un control de reintento explícito, y una carga liquidada responde a un solo clic abriendo `ImageLightbox` (los clics durante la carga se ignoran). El dimensionado sigue a DeepSeek Chat: la imagen única de un mensaje (`variant="single"`) se renderiza a 240px en su borde más largo con la relación de aspecto mostrada limitada a [0.25, 4] — el desbordamiento se recorta con `object-fit: cover`, anclado a la parte superior de las imágenes muy altas y a la izquierda de las muy anchas — y nunca supera su tamaño natural; una imagen entre varias (`variant="tile"`) es un cuadrado fijo de 64px. `ImageGallery` envuelve las imágenes de un mensaje en un grupo flex de ajuste alineado (`end` para mensajes de usuario, `start` para mensajes de asistente), elige la variante según el número de imágenes y no renderiza nada para una lista vacía. `ImageLightbox` es una vista previa modal a nivel de documento sobre la máscara de diálogo compartida (`--dsw-alias-bg-mask-1` + `--dsw-mask-blur`, pintada en su propia capa para que el desenfoque nunca toque la imagen previsualizada) que se cierra con Escape, al pulsar la máscara o con su control de cierre, y restaura el foco en su abridor al desmontarse.

## Superposición de soltar

`DropOverlay` es la invitación a pantalla completa que se muestra mientras un arrastre de archivo está sobre la página: ilustración, título y una línea de límites mientras los drops se aceptan (`disabled` intercambia la ilustración de bloqueado y oculta la línea de límites). La capa es pointer-inert — los listeners de arrastre a nivel de documento del dueño mantienen el recuento de entrada/salida y deciden aceptar/rechazar; la superposición solo muestra estado. Hace portal al body como el lightbox.

## Experiencia de modelo

Ninguna, porque el plugin solo renderiza el estado de adjuntos suministrado por la UI de conversación y no aporta ninguna entrada visible para el modelo.

#### Efecto en la caché KV

Ninguno; este paquete ni ensambla ni envía una petición al provider.

## Limitaciones conocidas y trabajo pendiente

- **Solo imágenes** — los archivos que no son imágenes no tienen todavía tarjeta de carril ni renderizador de historial; las tarjetas de archivo estilo DeepSeek Chat y los estados de progreso de subida esperan a que el composer acepte adjuntos que no sean imágenes.
- **Sin zoom ni descarga en el lightbox** — la vista previa renderiza el original solo al tamaño ajustado al viewport.
- **El lightbox no atrapa el foco** — establece `aria-modal` y restaura el foco al cerrar, pero Tab puede llegar a la página que hay detrás.
