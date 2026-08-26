# Adjuntos de imagen durables

[English](attachment.md) | Español

El seam de adjuntos separa la propiedad de las imágenes binarias del log de la sesión. Un productor entrega bytes codificados y validados a [`ctx.attachments`](#ctxattachments--attachmentstore-abstract-seam); el servicio publica una referencia inmutable direccionada por contenido solo después de que el objeto sea durable. Los eventos de sesión y los `ImageBlock`s visibles para el modelo contienen esa referencia y sus metadatos, nunca una object URL del navegador, una ruta temporal del host, una URL del provider ni un payload base64.

Los borradores no enviados del navegador pueden permanecer en memoria y los clientes nativos pueden dejarlos en el almacenamiento temporal del sistema operativo. En cuanto el host acepta un mensaje de usuario, sus imágenes se mueven a `<DSH_HOME>/attachments/v1` antes de que se añada el evento de usuario. La salida de imágenes estructurada del modelo sigue la misma regla de persistir antes que el evento.

Fuente: [`packages/attachment/attachment/src/types.ts`](../../packages/attachment/attachment/src/types.ts)

## Identidad y metadatos verificados

`AttachmentId` es una cadena opaca con marca (branded). El backend local emite actualmente `sha256:<digest>`, pero los consumers no deben analizar esa representación ni derivar de ella una ruta del sistema de archivos.

```ts type-equiv
/** Raster image formats accepted by the version-one attachment path. */
type ImageMediaType = 'image/png' | 'image/jpeg' | 'image/webp' | 'image/gif'
```

```ts type-equiv
/** Durable, serializable reference to one immutable normalized image. */
interface ImageAttachmentRef {
  /** Opaque storage identifier; never a filesystem path or bearer URL. */
  attachmentId: AttachmentId
  /** Media type verified from the stored bytes. */
  mediaType: ImageMediaType
  /** Exact encoded byte length. */
  bytes: number
  /** Intrinsic encoded width in pixels. */
  width: number
  /** Intrinsic encoded height in pixels. */
  height: number
  /** Optional display name stripped of local path information. */
  name?: string
  /**
   * Input dimensions after applying EXIF orientation and before normalization
   * scaling. Present only when normalization reduced the image.
   */
  originalDimensions?: {
    width: number
    height: number
  }
}
```

```ts type-equiv
/** Deployment-resolved limits used by upload admission and request buffering. */
interface ImageAttachmentLimits {
  maxImageBytes: number
  maxImagesPerMessage: number
  maxMessageImageBytes: number
  maxImagePixels: number
  /** Maximum intrinsic width and maximum intrinsic height in pixels for one image. */
  maxImageDimension: number
  mediaTypes: readonly ImageMediaType[]
}
```

El backend local admite como máximo 20 imágenes y 200 MiB de datos de origen codificados por mensaje. Una sola fuente puede usar hasta 20 MiB, 64.000.000 de píxeles y 8192 píxeles por cada lado. Estos límites de origen preceden a la etapa independiente de normalización, que limita el lado largo a 2048 píxeles y los datos codificados a 4 MiB por defecto.

La referencia registra las dimensiones intrínsecas y la longitud codificada para que los clientes puedan componer el historial sin decodificar antes, mientras que cada lectura autoritativa vuelve a comprobar contra el objeto el digest, la firma de medios, las dimensiones y los metadatos.

## Payloads de confirmación y de lectura verificada

```ts type-equiv
/** Base64-encoded image upload accompanying one wire request. */
interface EncodedImageAttachment {
  /** Declared media type, verified against the decoded bytes during admission. */
  mediaType: ImageMediaType
  /** Canonical base64 encoding of the image bytes. */
  data: string
  /** Optional display name; it is never interpreted as a path. */
  name?: string
}
```

```ts type-equiv
/** Request to validate and durably commit one image. */
interface SaveImageAttachment {
  data: Uint8Array
  /** Caller-declared media type, checked against fully decoded bytes. */
  mediaType: ImageMediaType
  /** Optional browser/provider display name; it is never interpreted as a path. */
  name?: string
}
```

```ts type-equiv
/** Stored image bytes returned after reference and digest verification. */
interface StoredImageAttachment {
  ref: ImageAttachmentRef
  data: Uint8Array
}
```

```ts type-equiv
/** Deterministic request-image policy selected by one exact model route. */
interface ImageRequestPolicy {
  /** Maximum width multiplied by height after aspect-preserving projection. */
  maxPixels: number
  /** Encoded-byte cap before base64 expansion or Files API upload. */
  maxBytes: number
}
```

```ts type-equiv
/** Cached request version derived from one provider-independent normalized attachment. */
interface RequestImageAttachment {
  /** Cache and upload-index key over the attachment id, policy, and fixed encoder parameters. */
  variantId: ImageVariantId
  /** Durable normalized attachment from which this request version was derived. */
  attachment: ImageAttachmentRef
  /** Encoded request bytes. */
  data: Uint8Array
  mediaType: ImageMediaType
  bytes: number
  width: number
  height: number
  /** Provider-compatible sample depth proven after request encoding. */
  depth: 'uchar'
  /** Provider-compatible color space proven after request encoding. */
  space: 'srgb'
  /** Whether the encoded request version retains an alpha channel. */
  hasAlpha: boolean
}
```

`saveImage()` prepara y confirma atómicamente un adjunto normalizado independiente del provider antes de devolver su `ImageAttachmentRef`. `saveImages()` prepara una sola vez cada adjunto validado antes de publicar el lote, de modo que el rechazo de validación no deja objetos parciales y la publicación no repite la decodificación ni la selección de calidad. `admitEncodedImages()` es la entrada de wire para las subidas base64 y delega en `saveImages()` la admisión por recuento, por bytes agregados y por lote ordenado. `readImage()` verifica un adjunto normalizado desde una ruta de sesión autorizada. `readImageRequest()` deriva y guarda en caché una versión de solicitud bajo un presupuesto exacto de píxeles y bytes de la ruta; las entradas nuevas se decodifican por completo antes de publicarse, mientras que los aciertos de caché usan una sonda de metadatos acotada. Los llamadores usan `Promise.all` sobre el método singular cuando necesitan un lote ordenado. La implementación local codifica de forma perezosa (lazy) los candidatos preferidos, aplica singleflight a las identidades de solicitud iguales, deja que cada llamada en espera cancele de forma independiente, detiene el trabajo compartido cuando no queda ninguna en espera y acota todas las transformaciones con su limitador a nivel de instancia, que por defecto permite dos transformaciones simultáneas. El servicio es neutro en retención: las sesiones reanudadas y bifurcadas (forked) pueden compartir objetos, por lo que la recolección de basura consciente de las referencias se difiere en lugar de ligarse a la eliminación de una sola sesión.

<!-- BEGIN GENERATED cordis-surface (gen-cordis-catalog.ts) — do not edit between markers -->

<a id="cordis-surface"></a>

## Cordis API

Generated from source by `scripts/gen-cordis-catalog.ts` (verified fresh by `pnpm run verify-cordis-catalog` in doc-sync; regenerate with `pnpm run gen-cordis-catalog`) — the language sides differ only in locale-specific paired document paths. Signature blocks use a `ts cordis-catalog` fence and keep the original source JSDoc; dispatch modes are defined in the [primer](../cordis-primer.es.md#dispatch-modes), and the framework-inherited `ctx` API lives in [cordis-api/inherited.md](../cordis-api/inherited.es.md).

<a id="ctxattachments--attachmentstore-abstract-seam"></a>

### `ctx.attachments` — `AttachmentStore` (abstract seam)

Immutable binary attachment service. Implementations validate bytes before publishing a reference.

```ts cordis-catalog
/**
 * Validate one image without persisting it.
 * Batch callers validate every member before saving any member.
 * @param input - encoded bytes, declared media type, and optional display name.
 * @returns completion after the encoded raster has been fully decoded.
 */
abstract validateImage(input: SaveImageAttachment): Promise<void>

/**
 * Validate and durably commit one ordered image batch.
 * @param inputs - encoded images in owning-message order.
 * @returns durable normalized attachment references in the same order after every member succeeds.
 */
async saveImages(inputs: readonly SaveImageAttachment[]): Promise<readonly ImageAttachmentRef[]>

/**
 * Validate and durably commit one image before its owning session event is appended.
 * The returned reference describes the persisted normalized image. When
 * normalization reduces the raster, its `originalDimensions` records the
 * orientation-applied input dimensions.
 * @param input - encoded bytes, declared media type, and optional display name.
 * @returns the durable content-addressed normalized image reference.
 */
abstract saveImage(input: SaveImageAttachment): Promise<ImageAttachmentRef>

/**
 * Read one image and verify that bytes still match the recorded reference.
 * @param ref - durable reference from the session log.
 * @param signal - optional cancellation for backend read and verification work.
 * @returns the verified bytes and normalized attachment reference.
 * @throws the signal reason when aborted, or a storage error when verification fails.
 */
abstract readImage(ref: ImageAttachmentRef, signal?: AbortSignal): Promise<StoredImageAttachment>

/**
 * Generate or read one deterministic model-request version from the stored normalized image.
 * @param ref - durable provider-independent normalized attachment reference.
 * @param policy - exact route pixel and encoded-byte budget.
 * @param signal - optional cancellation.
 * @returns request bytes and the cache/upload identity covering every transform input.
 */
readImageRequest( ref: ImageAttachmentRef, policy: ImageRequestPolicy, signal?: AbortSignal, ): Promise<RequestImageAttachment>
```

Source: [`packages/attachment/attachment/src/index.ts`](../../packages/attachment/attachment/src/index.ts)
<!-- END GENERATED cordis-surface -->
