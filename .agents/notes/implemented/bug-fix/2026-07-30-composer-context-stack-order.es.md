# Agent Note: Orden de la pila de contexto del composer

Status: implemented

English | [中文](2026-07-30-composer-context-stack-order.zh.md)

## Problema

Goal, Todo y Queue contribuyen de forma independiente a la misma lista `conversation.input.dock`, pero su orden de registro y reglas de espaciado no codificaban la matriz de composición. El renderer colocaba por tanto Todo antes de Queue y Goal, mientras Queue y Goal llevaban márgenes negativos pensados para el límite del composer. Con los tres presentes, Queue se unía a Goal y Goal se unía al composer, invirtiendo la jerarquía del diseño.

## Decisión

La [decisión de alineación Todo-primero](2026-08-02-todo-first-composer-context-order.es.md) posee el orden ascendente actual. Esta nota retiene el contrato de pila alrededor de ese orden: los huecos numéricos dejan sitio a que futuras entradas declaren su posición pretendida sin depender del orden de activación de plugins, y la barra del composer sigue a la lista.

`ConversationRoot` posee los 6px de espacio entre tarjetas de contexto independientes. Goal es una tarjeta independiente de 752×36px y el Todo colapsado es una tarjeta independiente de 752×44px. Queue es la entrada terminal del dock: su envolvente de 776px contiene la misma columna de panel de 752px y resta el hueco compartido más un solape de layout con nombre de 5px, así que la tarjeta del composer posterior pinta solo sobre el borde de la cola. Las entradas vacías renderizan null y no consumen hueco.

El orden y el solape son contratos separados. El orden de registro establece la jerarquía semántica; las variables CSS sobre la pila establecen la geometría compartida. Queue no infiere que puede solapar meramente por ser la última entrada visible, porque Goal o Todo pueden ser la última tarjeta de contexto visible cuando no hay cola y deben permanecer separadas del composer.

## Verificación

Los tests de registro fijan los tres valores de orden. El escenario keyless de Queue en browser renderiza Todo, Goal y Queue juntos, fija su orden de accesibilidad y comprueba sus bordes visibles de tarjeta; los escenarios enfocados de Goal y Queue cubren sus estados independientes.

## Alternativas consideradas

**Conservar márgenes negativos independientes en Goal y Queue.** Descartado porque el vecino afectado cambia con el orden de slots; un margen local no puede expresar qué relación está permitida salvo que el orden semántico también quede fijo.

**Renderizar cada id de dock conocido por separado en `ConversationRoot`.** Descartado porque convierte un slot de lista extensible en un inventario hardcodeado de componentes y fuerza al propietario a cambiar por cada nuevo registrante.

**Meter debajo la entrada del dock que quede última.** Descartado porque Goal y Todo son tarjetas independientes. Su matriz de ausencias no debe cambiar la semántica de superficie de la tarjeta que permanezca.

## Consecuencias

La jerarquía visual es estable para cada combinación de presencia, y Queue es la única superficie de contexto unida al composer. Los nuevos plugins de input-dock deben elegir un orden relativo a Todo `0`, Goal `10` y Queue `20`; una entrada después de Queue exige además una decisión explícita sobre qué superficie posee el límite del composer.
