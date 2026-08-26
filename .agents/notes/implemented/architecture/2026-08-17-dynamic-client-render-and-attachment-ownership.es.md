# Agent Note: Render de cliente dinámico y titularidad de los adjuntos

Status: implemented

[English](2026-08-17-dynamic-client-render-and-attachment-ownership.md) | Español

## Problema

El grafo de cliente escrito por el host gobierna los plugins de navegador, pero tres rutas de presentación quedaban fuera de ese ciclo de vida. El kernel web creaba la raíz React y una pseudo-entrada de ensamblaje propiedad del shell, `ui-conversation` importaba los componentes de adjuntos como valores de paquete, y el shell importaba los estilos globales de ui-theme. Desactivar, fallar o recargar un plugin no gobernaba por tanto todo el renderizado y el CSS que le pertenecían.

La página de carga y de fallo tiene el requisito opuesto: debe seguir siendo utilizable cuando cualquier plugin dinámico, incluido el renderizador, no logra activarse. No puede depender del árbol React cuyo fallo reporta.

## Decisión

`@deepseek-ai/dsh-client-web` es un kernel de arranque sin framework. Dibuja su página de carga y de fallo con operaciones DOM y respaldos CSS locales, construye el sistema de módulos del cliente y el Loader de Cordis, crea la entrada de arranque de módulos adoptada estáticamente más toda entrada del grafo del host, y espera hasta que toda fibra esté ACTIVE. Los cambios de estado del Loader conservan un único nodo spinner y actualizan solo su arco CSS cuando una entrada pasa a activa por primera vez. El arco crece de un quinto a cuatro quintos del anillo, preservando un hueco visible durante toda la rotación. Tras asentarse el elenco, el kernel resuelve `ctx.uiRenderer` y entrega el contenedor existente a `mount()`.

`@deepseek-ai/dsh-client-ui-renderer` es un plugin de cliente dinámico `immediately`. Es dueño de los outlets de slot React, de SessionProvider y del enlace observable-a-uSES. Tras activarse sus inyecciones `slots` y `sessions`, instala el renderizador de slots y suministra `ctx.uiRenderer`. `mount()` hidrata el DOM de arranque escrito por el kernel y después lo reemplaza con la aplicación ensamblada en un efecto de layout antes de que el navegador pueda pintar un frame intermedio. El nodo spinner hidratado conserva su fase de animación. El árbol ensamblado proyecta el título de la sesión seleccionada y realiza la única llamada `renderSlot('root')` a nivel de contexto. El servicio, la instalación del renderizador y la raíz React se disponen todos junto con sus dueños.

`ui-conversation` declara `conversation.input.attachments` y `conversation.message.images` y suministra los datos de adjuntos, los callbacks, la carga autorizada de imágenes y su asiento de locale. `ui-attachment` espera esas declaraciones a través de `ctx.slots.inject()` y registra el carril/soltadero del borrador y la galería/lightbox histórica de imágenes. Las implementaciones React siguen siendo valores internos del paquete; la composición entre plugins usa slots. Esta integración de paquete supera la regla de import directo de la [nota de visualización de adjuntos](../feature/2026-08-11-web-attachment-display-alignment.es.md) sin cambiar las decisiones visuales y de interacción de esa nota.

ui-theme importa sus cinco hojas de estilo globales como cadenas `?inline`. Su entrada de cliente llama a `installThemeStyles(ctx)`, que instala una etiqueta de estilo por hoja a través de `ctx.effect()`, de modo que descargar o recargar ui-theme retira o reemplaza su CSS global con el mismo ciclo de vida que su servicio. El kernel web conserva solo los valores por defecto de montaje y una paleta autocontenida de la página de arranque cuyas fuentes y colores coinciden con los tokens de tema correspondientes.

React, React DOM, Cordis, ui-slots y ui-primitives siguen siendo módulos de plataforma estáticos con una identidad de navegador. El bundle dinámico ui-renderer consume esos módulos compartidos y es dueño de los efectos de renderizado.

## Verificación

Las pruebas de componentes fijan el spinner de progreso persistente, la hidratación sin mutación del DOM de arranque, el título del documento, el árbol de aplicación, las entradas de adjuntos y la disposición. El arranque del bundle compilado ensamblado ejercita la tabla de módulos real y las entradas dinámicas, mientras que las pruebas de estilos de tema demuestran que sus etiquetas se instalan y disponen con la fibra del plugin. El carril de replay del navegador cubre la entrega completa desde la página sin framework hasta la aplicación renderizada.

## Alternativas consideradas

**Conservar la pseudo-entrada de ensamblaje propiedad del shell.** Rechazado porque permanece invisible al grafo del host y convierte el render en un camino Loader especial aunque el ensamblaje tenga dependencias de servicio y efectos de ciclo de vida ordinarios.

**Conservar los átomos de adjuntos exportados e importarlos desde ui-conversation.** Rechazado porque un import directo de componente sortea la composición independiente de plugins y la titularidad de la recarga. Los datos del dueño siguen viajando directamente a través de props de slot tipadas; solo la selección de presentación es dinámica.

**Conservar los estilos de ui-theme en la hoja de estilos base del shell.** Rechazado porque el CSS del tema seguiría activo cuando el plugin de tema estuviera ausente o fallara y no participaría en la limpieza de recarga de plugins.

**Renderizar la página de fallo con React.** Rechazado porque un fallo de ui-renderer o del árbol React no debe retirar el único diagnóstico disponible en el navegador.

## Consecuencias

El grafo del host contiene todo dueño de render dinámico, y el HMR reemplaza la presentación de adjuntos, el ensamblaje de render y el CSS del tema a través del ciclo de vida de plugins. Un fallo de ui-renderer deja una página de fallo DOM legible en lugar de un montaje React en blanco. Omitir ui-attachment deja deliberadamente vacíos sus slots opcionales; la composición web entregada lo incluye, y una entrada configurada que falle la activación impide la entrega de la aplicación completa.

La aplicación sigue esperando al elenco completo del cliente antes de su primer frame React. El shell sigue empaquetando estáticamente las identidades de los módulos de plataforma, y la página de arranque mantiene una paleta clara/oscura privada pequeña porque el CSS de ui-theme no está disponible hasta que ese plugin se materializa.
