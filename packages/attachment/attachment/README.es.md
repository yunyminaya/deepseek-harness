# @deepseek-ai/dsh-attachment
[English](README.md) | Español

El seam de adjuntos duraderos. `ctx.attachments` valida y confirma de forma duradera una imagen normalizada independiente del provider, y devuelve un `ImageAttachmentRef` serializable; los consumidores nunca persisten rutas de navegador, URLs de objetos, URLs de provider ni base64 en los eventos de sesión.

Las imágenes del compositor sin enviar siguen siendo borradores temporales propiedad del navegador. `validateImage` ejecuta la política de admisión completa sin persistir. `saveImages` es dueño de los límites de recuento del lote y de bytes agregados, prepara cada adjunto normalizado antes de publicar cualquier miembro, luego confirma en orden y devuelve referencias solo después de que el lote completo tenga éxito. Un fallo de almacenamiento posterior no devuelve referencias parciales, aunque un objeto inmutable anterior con direccionamiento por contenido puede quedar inalcanzable hasta que exista una recolección de basura consciente de referencias. `AttachmentError.code` usa la unión de cadenas cerrada `AttachmentErrorCode`. Su subconjunto `ImageAdmissionErrorCode` marca los fallos de entrada de imagen corregibles por el llamador; `isImageAdmissionError` reconoce ese subconjunto en runtime para que cada adaptador de protocolo pueda mapear su propio vocabulario de errores. `saveImage` confirma una imagen aceptada antes de publicar cualquier evento de sesión visible para el modelo y devuelve su `ImageAttachmentRef`. Cuando la normalización reduce el ráster, la referencia registra en `originalDimensions` el tamaño de entrada con la orientación aplicada. `readImage` verifica el adjunto normalizado contra sus metadatos registrados. `readImageRequest` deriva de forma determinista una versión de petición con el tamaño de la ruta, cuya identidad cubre el id del adjunto, la versión de transformación, los presupuestos de píxeles y bytes, y los ajustes del codificador. Los llamadores componen lotes ordenados con `Promise.all(refs.map(...))`; la implementación local sigue acotando la compresión mediante su limitador de instancia, su caché y su singleflight. Los llamadores pueden cancelar lecturas y proyecciones; las implementaciones preservan la cancelación en lugar de traducirla en un fallo de almacenamiento.

`admitEncodedImages(attachments, images)` es la entrada de cable compartida que usa todo endpoint RPC que acepta subidas del navegador (el endpoint de prompt de sesión y el ejecutor de comandos): impone base64 canónico en cada miembro y luego delega la admisión del lote — límites, validación, confirmación ordenada — en `saveImages`. La forma de subida base64 es `EncodedImageAttachment`, exportada desde `@deepseek-ai/dsh-attachment/types` para que los contratos de cable puedan referenciarla.

## Experiencia de modelo

Indirectamente, mediante el `ImageBlock` central neutro respecto al rol y los adaptadores de provider que resuelven su referencia duradera en una versión de petición exacta. Los descriptores de petición exponen el id completo del adjunto y las dimensiones reales de la petición.

#### Efecto de KV Cache

Añadir una imagen cambia la petición al provider y por tanto invalida el sufijo de petición afectado.

## Limitaciones conocidas y trabajo diferido

- La versión uno acepta solo PNG, JPEG, WebP y GIF.
- La retención y la recolección de basura quedan diferidas porque las sesiones reanudadas y bifurcadas pueden compartir objetos inmutables.
- Los archivos genéricos, el audio, el vídeo y los borradores persistentes sin enviar requieren contratos separados de ciclo de vida y de provider.
