# Agent Note: Orden de contexto del composer con Todo primero

[English](2026-08-02-todo-first-composer-context-order.md) | [中文](2026-08-02-todo-first-composer-context-order.zh.md) | Español

Status: implemented

## Problema

La pila de contexto del composer renderizaba Goal antes que Todo aunque el diseño del Harness ordena el plan de tarea actual antes de su objetivo en curso y de la Queue pendiente. Todo también usaba el ancho de 776px del wrapper de Queue como su ancho de tarjeta visible, mientras que Goal y el panel de Queue renderizaban en la columna de tarjeta compartida de 752px. El resultado invertía la jerarquía de información prevista y dejaba a Todo más ancho que ambos paneles adyacentes.

## Decisión

La lista `conversation.input.dock` usa un único orden de producto ascendente: Todo en `0`, Goal en `10` y Queue en `20`, seguidos de la barra de composer fuera de la lista. El orden de registro sigue siendo la fuente semántica de verdad; el renderizador no fija en código los ids de componentes conocidos ni repara su orden con CSS.

Todo, Goal y el panel visible de Queue comparten la columna de tarjeta de 752px dentro del tope de 800px del composer. Queue conserva un wrapper de 776px con un inset transparente de 12px a cada lado porque ese wrapper es el propietario del solapamiento con el composer. Todo es una tarjeta independiente y no un wrapper, por lo que su ancho responsivo y su ancho máximo restan ambas capas de inset directamente. Goal usa la misma columna responsiva y limita su barra interna a 752px, conservando bordes coincidentes por debajo del tope de escritorio.

El [contrato de la pila del composer](2026-07-30-composer-context-stack-order.es.md) sigue siendo el propietario del espaciado entre tarjetas y del solapamiento exclusivo de Queue con el composer. Esta decisión sustituye solo el orden de Goal-primero de esa nota.

## Verificación

Las pruebas de registro de Todo y Goal fijan los órdenes `0` y `10`; Queue sigue fijado en `20`. El escenario de navegador sin clave de Queue renderiza los tres paneles de forma concurrente, registra su orden de accesibilidad Todo–Goal–Queue y compara sus bounding boxes visibles en la línea base de escritorio de 1680px y en un viewport sub-tope de 640px antes de ejercitar las mutaciones de Queue.

## Alternativas consideradas

**Reordenar los paneles conocidos dentro de `ConversationRoot`.** Rechazado porque `conversation.input.dock` es una lista ordenada extensible; un inventario de componentes fijado en código haría discrepar el orden de activación de plugins del orden renderizado.

**Usar `order` de CSS para mover Todo visualmente.** Rechazado porque el orden de accesibilidad y de teclado debe coincidir con la jerarquía visual, y el libro mayor de slots ya es el propietario del orden semántico.

**Mantener Todo con el ancho del wrapper de Queue.** Rechazado porque el inset transparente del wrapper de Queue es infraestructura de layout para su solapamiento con el composer, no parte de la columna de panel visible.

## Consecuencias

El plan de tarea vigente aparece antes del objetivo en curso, el trabajo pendiente de Queue sigue siendo el más cercano al composer, y las tres tarjetas visibles comparten un único borde horizontal. Los futuros plugins de input-dock eligen una posición explícita relativa a Todo `0`, Goal `10` y Queue `20`; solo Queue es el propietario del solapamiento terminal del wrapper.
