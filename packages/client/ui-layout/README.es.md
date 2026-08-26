# @deepseek-ai/dsh-client-ui-layout

[English](README.md) | [中文](README.zh.md) | Español

Plugin de carcasa: AppFrame de tres columnas (asas de arrastre y cadena de concesión) más el servicio de geometría de paneles `ctx.layout`; se registra en el slot `root` propiedad del runtime y declara `sidebar`, `conversation`, `details` y `conversation.empty`. El límite de redimensión de la barra lateral es una franja de acierto invisible, mientras que el límite de details conserva su píldora flotante; solo details se encoge durante la concesión y luego se auto-cierra. Una barra lateral cerrada conserva un carril de control de 56px mientras que details se cierra a ancho cero. El paquete también asienta el presentador de tema: consume instantáneas resueltas de `ctx.theme` y las proyecta sobre el documento (`html { color-scheme }` para el chrome nativo del UA, `body[data-ds-dark-theme]` desde el esquema de color activo, los tokens de alias del tema como variables en línea sobre body, y un `<meta name="theme-color">` propio cuyo contenido sigue el fondo de body calculado). Medir después de aplicar la paleta y los tokens mantiene el fondo renderizado como la única autoridad de color; disponer del presentador elimina su nodo de metadatos junto con sus otras escrituras globales.

AppFrame monta siempre las columnas de conversación y details; una Session conectada se renderiza a través de `SessionProvider`. El almacén de diseño transitorio inicia la barra lateral en su ancho por defecto y details cerrado, y nunca lee ni escribe `localStorage`. El hero y otros estados no seleccionados derivan también un ancho de details renderizado de cero sin cambiar esa preferencia almacenada. AppFrame conserva el último id de Session no en blanco a través de esos estados: la primera Session permanece cerrada, una acción explícita de details abre el ancho por defecto del contrato, volver a la misma Session restaura su ancho sin cambios, y seleccionar una Session distinta cierra details antes del pintado. La cuota de propiedad del dueño de la conversación está vacía, mientras que la cuota de propiedad del dueño de la barra lateral contiene solo `collapsed` y `width`; los registrantes obtienen los datos de negocio de hooks estándar y las acciones de sus propias caras de inject.

Los exports de `/client` son el cuerpo del plugin (`apply`/`inject`), `LayoutController` y las cuatro interfaces de cuota de propiedad. AppFrame, el almacén de paneles y el solucionador de concesión permanecen internos al paquete.

## Experiencia de modelo

Ninguna, ya que la carcasa de diseño gestiona el estado de visualización del navegador; nada aquí llega a una solicitud de modelo.

#### Efecto en KV Cache

Ninguno; este paquete no ensambla ni envía ninguna solicitud de provider.

## Limitaciones conocidas y trabajo diferido

- **La geometría de los paneles es transitoria** — recargar restaura la barra lateral por defecto y details cerrado; cambiar entre ids de Session distintos también cierra details y olvida su ancho arrastrado, mientras que las superficies no seleccionadas renderizan details a ancho cero sin modificar la geometría.
- **El auto-cierre de la cadena de concesión deriva un ancho cero sin tocar el ancho preferido** — el panel se restaura solo cuando la ventana se ensancha; los consumidores no deben leer el ancho de details almacenado como la verdad renderizada.
- **Sin anclaje de scroll durante el reflujo por compresión** — los cambios de diseño pueden mover el viewport del lector.
