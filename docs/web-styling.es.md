# Referencia de estilos de la interfaz web

[English](web-styling.md) | Español

Esta referencia define la titularidad de los estilos y las reglas de componentes para los paquetes de cliente de navegador. Los valores de token actuales viven en [`packages/client/ui-theme/src/styles/`](../packages/client/ui-theme/src/styles/); este documento no duplica ese inventario generado a partir del código fuente.

## Titularidad

[`ui-theme`](../packages/client/ui-theme/README.es.md) es el dueño de la escala estática `--dsw-*`, los alias semánticos, la tipografía, el movimiento, los degradados, las sombras, los estilos de barra de desplazamiento y la preferencia claro/oscuro. [`ui-layout`](../packages/client/ui-layout/README.es.md) aplica la instantánea resuelta del tema al documento. Los paquetes de funcionalidad consumen alias semánticos y no definen otro tema global.

Las hojas de estilo globales pertenecen a `ui-theme/src/styles/`. Los estilos de componentes viven junto a su componente como CSS Modules. Un componente puede definir una propiedad personalizada local cuando su valor forma parte del contrato de maquetación o presentación de ese componente; los colores, la tipografía, la elevación y el movimiento compartidos pertenecen al paquete de tema.

## Reglas de componentes

- Usa CSS Modules y `clsx`; no añadas una biblioteca de componentes ni Tailwind.
- Usa los tokens semánticos `--dsw-alias-*` en los componentes de funcionalidad. No copies valores estáticos de la paleta ni escribas colores literales allí.
- Mantén los selectores de tema fuera del CSS de los componentes de funcionalidad. Las anulaciones claro/oscuro pertenecen al dueño del tema.
- Empareja los tamaños de fuente con las alturas de línea y usa las variables de tipografía del tema cuando un rol existente coincida.
- Mantén sin ajustar el texto de origen, la salida de terminal y las líneas de diff cuando su contrato de componente exija conservar las columnas; usa los estilos de barra de desplazamiento compartidos en lugar de selectores de barra específicos del componente.
- Pon la presentación en CSS. Los estilos React en línea pueden pasar valores de propiedades personalizadas locales del componente, pero no deben codificar ramas del tema.
- Conserva la visibilidad del foco de teclado y el comportamiento de movimiento reducido al añadir transiciones o controles solo de hover.

## Cómo cambiar el sistema

Añade o cambia un token compartido en la hoja `ui-theme` propietaria y luego consume su alias semántico desde los paquetes de funcionalidad. Actualiza la referencia del paquete propietario cuando cambie un contrato público de estilos. El comportamiento visual sigue la [política de pruebas](testing.es.md); la [Agent Note del sistema de estilos](../.agents/notes/implemented/process/2026-07-19-web-styling-system.es.md) registra la justificación del framework.
