# Agent Note: Metadatos de color de tema resueltos

Status: implemented

[English](2026-08-06-resolved-theme-color-metadata.md) | Español

## Problema

El cliente web puede resolver su tema independientemente de la preferencia del sistema operativo, así que un único `theme_color` de manifest o metadatos estáticos con media queries pueden discrepar de una selección explícita de Claro u Oscuro. El chrome del navegador alrededor de una página instalada u ordinaria puede entonces no coincidir con la superficie de la app aunque el presentador de layout ya sea dueño de la paleta documental resuelta.

## Decisión

El `ThemePresenter` de ui-layout es dueño de un `<meta name="theme-color">` junto a su `color-scheme` raíz, atributo de paleta oscura y escrituras de tokens en línea. Tras aplicar la paleta y los overrides de tokens de un snapshot resuelto, el presentador lee el `background-color` calculado del body en el elemento de metadatos e inserta ese único nodo en el head del documento. Los snapshots posteriores actualizan el mismo nodo, y el disposal lo elimina.

El fondo del body renderizado sigue siendo la autoridad de color. El manifest PWA no lleva ningún `theme_color` o `background_color` estático, y `ThemeDefinition` no gana ningún segundo campo de color que pudiera derivar de la paleta de tokens. Esto permite también que el token de fondo base de un tema registrado llegue a la UI del navegador por el mismo camino de aplicación que su superficie de página.

## Verificación

El contrato unitario del presentador cubre los colores calculados claro y oscuro, la reutilización del nodo y el disposal. El test de composición de ui-layout cubre la inserción inicial, la reutilización dirigida por eventos y la limpieza de fiber. El escenario de ajustes del navegador Web conduce Claro, Oscuro, Sistema, cambios del sistema operativo y recarga a través de la composición incluida, afirmando un elemento de metadatos cuyo contenido es igual al fondo del body calculado sin errores de consola. El cambio de metadatos no tiene salida de árbol de accesibilidad renderizada, así que el golden existente del escenario permanece sin cambios.

## Alternativas consideradas

**Poner `theme_color` en el manifest.** Un manifest proporciona un único valor para toda la app, así que cualquier paleta integrada puede discrepar de él; el manifest omite deliberadamente el campo.

**Declarar metadatos claro y oscuro con media queries `prefers-color-scheme`.** Las media queries siguen al sistema operativo, no a una selección explícita dentro de la app, y por tanto no pueden representar la preferencia resuelta.

**Añadir un campo `themeColor` a cada `ThemeDefinition`.** Un valor separado da a los temas personalizados una elección independiente de chrome de navegador, pero duplica el color de fondo base y permite que la página y la UI circundante deriven. Un campo distinto puede introducirse si un tema soportado necesita esa diferencia intencional.

## Consecuencias

Los navegadores que lo soportan actualizan la UI circundante después de que el cliente aplique su snapshot resuelto inicial y tras cada cambio de tema; los navegadores sin soporte de `theme-color` ignoran los metadatos. Como el valor viene de la presentación calculada, el cliente debe mantener un fondo de body concreto. El presentador crea y elimina su propio nodo, mientras que los metadatos de head no relacionados permanecen intactos.
