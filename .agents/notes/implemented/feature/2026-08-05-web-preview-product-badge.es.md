# Agent Note: Insignia de vista previa del producto en la Web

Status: implemented

[English](2026-08-05-web-preview-product-badge.md) | Español

## Problema

El estado vacío de la Web no identifica el producto como vista previa. Los usuarios pueden entrar en la superficie principal de sesión sin ver que el producto es preliminar, mientras que un ajuste de despliegue malrepresentaría una decisión de ciclo de vida de todo el producto como una elección del operador.

## Decisión

El hero vacío renderiza siempre una insignia localizada `Preview` / `预览版` bajo el titular. No tiene interruptor de configuración: el estado de vista previa es una identidad de producto compartida por todos los despliegues, no un parámetro que varíe por despliegue.

La insignia conserva el fondo business-tertiary para que ambos temas mantengan el contexto azul de producto, y usa el token de etiqueta primary del tema para el texto. Esa combinación da a un texto normal de 12px contraste suficiente en los temas claro y oscuro; el foreground business-primary se reserva para acentos más grandes o no textuales porque no alcanza el contraste requerido sobre este fondo.

La insignia abandona el producto cuando el primer release etiquetado elimina la postura preliminar del repositorio, o cuando la decisión de producto dueña declara completada la fase de vista previa. Ese cambio elimina la insignia y su clave de locale juntas, en lugar de añadir un interruptor de runtime.

## Alternativas consideradas

**Hacer configurable el estado de vista previa.** Rechazado porque dos despliegues del mismo producto preliminar no deben presentar identidades de ciclo de vida distintas, y un campo de configuración convertiría el estado de release del producto en una elección de operador no soportada.

**Usar texto business-primary sobre el fondo business-tertiary.** Rechazado porque el contraste resultante en los temas claro y oscuro queda por debajo del requisito 4.5:1 para el texto de 12px de la insignia.

**Ocultar la insignia del árbol de accesibilidad.** Rechazado porque el estado de vista previa es información de producto, no decoración; el titular accesible incluye por tanto el texto de la insignia.

## Consecuencias

Cada sesión nueva expone la misma identidad de vista previa localizada en la salida visual y de accesibilidad. Eliminar el estado de vista previa es una edición explícita de release de producto, y la insignia favorece texto neutro legible sobre un tratamiento todo-azul, conservando el fondo con tinte de producto.

## Pruebas

El test del componente de conversación cubre ambos valores localizados de la insignia, y los snapshots del ciclo de vida Web fijan la insignia en inglés en el hero vacío ensamblado.
