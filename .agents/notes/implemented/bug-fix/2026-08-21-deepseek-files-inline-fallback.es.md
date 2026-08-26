# Agent Note: Recuperar las peticiones de imagen de DeepSeek de los fallos de resolución de Files

Status: implemented

[English](2026-08-21-deepseek-files-inline-fallback.md) | Español

## Problema

La ruta de visión directa de DeepSeek usa ids de archivo del provider para que las peticiones repetidas no reenvíen los bytes de la imagen. Un endpoint Files no disponible, no soportado o estancado puede impedir el chat antes de que comience la petición al modelo, aunque el mismo endpoint siga aceptando datos de imagen inline. Un fallback que conservara el presupuesto de Files de 128MiB excedería el límite del cuerpo de la petición inline, mientras que un fallback que transformara las imágenes de forma independiente podría enviar píxeles distintos de los del intento con id de archivo fallido.

## Decisión

Files sigue siendo el transporte preferido. Cada resolución de archivo de imagen de petición tiene el plazo configurable `filesApiTimeoutMs`, un minuto por defecto. El plazo de inactividad del stream se fija por defecto en cinco minutos, así que el plazo de Files normalmente deja tiempo para el fallback inline. Un despliegue puede configurar el plazo de inactividad del stream para que expire primero. Las resoluciones exitosas refrescan el watchdog de inactividad exterior. La cancelación del llamador y el plazo exterior del stream siguen siendo resultados terminales.

Un fallo de resolución de archivo descarta las partes de archivo transitorias ensambladas para ese intento de chat y reconstruye la petición de imagen completa con data URLs base64. Cada imagen conservada usa el `RequestImageAttachment` determinista ya preparado; el fallback no realiza ningún decode, resize o encode adicional, y una petición de chat nunca mezcla ids de archivo con imágenes inline. Los mapeos de subida confirmados antes de que falle una imagen posterior siguen disponibles para peticiones posteriores. La siguiente petición vuelve a intentar Files, así que la recuperación no requiere ningún estado de caída a nivel de proceso.

El fallback inline tiene un high watermark separado expandido por base64, `maxInlineRequestImageBytes`, de 20MiB por defecto. `inlineImageOffloadByteQuantum` se fija por defecto en 10MiB, de modo que cruzar el high watermark avanza el prefijo determinista de imagen más antigua hasta el siguiente límite de eliminación de 10MiB. El límite existente de 600 imágenes y el cuanto de recuento siguen aplicándose. El modo de archivo conserva su high watermark de 128MiB y su cuanto de eliminación de 64MiB.

Los errores de chat del provider conservan sus clasificaciones existentes. Un id de archivo obsoleto se invalida, se vuelve a subir y se reintenta una vez. Si esa resolución de reemplazo falla, el reintento permitido usa la representación inline. Un fallo de chat genérico no cambia de transporte porque no establece que la resolución de Files fallara.

## Alternativas consideradas

**Enviar primero las imágenes inline.** Rechazado porque las subidas exitosas a Files permiten reutilizar bytes de petición deterministas entre turnos sin repetir el base64 en cada petición.

**Mezclar ids de archivo resueltos con imágenes inline después de que una subida falle.** Rechazado porque la petición seguiría dependiendo del servicio Files que falla y tendría dos presupuestos de imagen independientes.

**Aplicar el límite de Files de 128MiB al fallback inline.** Rechazado porque el base64 expande el payload y puede exceder el límite del cuerpo de la petición de chat. El presupuesto de 20MiB deja espacio para JSON, historial de texto y herramientas.

**Recordar la caída y omitir Files en peticiones posteriores.** Rechazado porque un estado de circuito local al proceso introduce el momento de la recuperación y el estado de fallo compartido. Reintentar Files en la siguiente petición detecta la recuperación del servicio sin otro temporizador.

## Verificación

Las pruebas del serializer cubren las representaciones de archivo y data-URL sobre las mismas versiones de petición, todos los tipos de medio soportados, la colocación de resultados de herramienta y el offload base64 de 20 a 10. Las pruebas del adapter cubren el fallo inmediato de resolución, el fallo tras un conjunto parcial de ids de archivo, el fallback disparado por plazo, el fallo de reemplazo de id obsoleto, los cuerpos de petición totalmente inline, la cancelación del llamador sin fallback y el fallo de chat genérico sin cambio de transporte. Las pruebas de configuración cubren ambos límites inline y los plazos independientes de Files y de inactividad del stream.

## Consecuencias

Una caída de Files ya no impide un chat de imagen que quepa en el presupuesto inline. El fallback repite los bytes de imagen y puede omitir más historial que el modo de archivo porque su límite es menor. Una petición puede dejar subidas exitosas atrás cuando falla una imagen posterior, pero sus mapeos indexados son reutilizables y no cambian el cuerpo de chat que envía el fallback. Las operaciones explícitas de gestión de archivos siguen exponiendo sus propios fallos.
