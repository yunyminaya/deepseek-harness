# Agent Note: La guía del creador aterriza como una señal de introducción en el chip de preset

Status: implemented

[English](2026-08-10-creator-guidance-introduce-cue.md) | Español

## Problema

Crear un preset ocurre dentro de una sesión de modo Creator, pero la sección de ajustes no daba ninguna vía hacia ese hecho. La entrada de creador quedaba fuera de los grupos del elenco, el grupo personalizado desaparecía por completo mientras no tuviera ningún miembro, y hacer clic en la entrada llevaba al usuario a la pantalla de nueva sesión sin nada que marcara qué había cambiado: el chip de preset escenificado se renderizaba exactamente como si el usuario lo hubiera elegido a mano. Los usuarios informaron no entender que el flujo se había movido, o que la sesión que estaban a punto de iniciar era el lugar donde se construye el preset (#2184).

## Decisión

El grupo personalizado permanece en pantalla mientras está vacío — encabezado más la entrada de creador, que vive dentro del grupo como la invitación permanente «tu preset aparecerá aquí» en lugar de flotar bajo el elenco.

Una elección escenificada desde otra pantalla porta una bandera `introduce` de un solo disparo a través del store de asiento (`stage(id, introduce)`), y el chip la anuncia: el icono del preset aparece con un ease-in durante 150ms, y después los caracteres del nombre se desvanecen hacia arriba con un escalonado en el momento en que el icono aterriza. El escalonado está limitado dos veces — 40ms por tick para nombres CJK cortos, y una ventana de revelado compartida de 200ms (`min(40, 200/(n-1))`) para que un nombre latino largo termine en el mismo tiempo que su contraparte CJK en lugar de alargar la ejecución por carácter. El CSS es dueño del movimiento; el componente lo arma y reconoce la señal una vez que la ejecución termina, así que la bandera nunca se reproduce en un montaje posterior. `prefers-reduced-motion` y un nombre de visualización vacío reconocen de inmediato sin ejecución.

La señal es presentación pura: es estado del store de asiento del lado del cliente, nunca un evento de sesión, porque la composición visible al modelo ya la porta el propio preset escenificado.

## Alternativas consideradas

**Un toast o callout en la pantalla de nueva sesión.** Explica más, pero no apunta a nada — el chip es el artefacto que el usuario debe volver a encontrar después, y una caja descartable enseña la caja, no el control. La señal pone el movimiento en el propio control.

**Un tick fijo por carácter.** La primera implementación usaba 60ms por carácter incondicionalmente; un nombre de preset en inglés tardaba más de tres veces lo que su contraparte china de cuatro caracteres, leyéndose como lag en lugar de énfasis. La ventana de revelado compartida hace que la duración sea una propiedad de la señal, no de la locale.

**Animar la elección dentro del diálogo de ajustes antes de salir.** El diálogo se cierra como parte del gesto — salir de los ajustes es como el flujo dice que el trabajo ocurre en la sesión — así que cualquier cosa reproducida ahí se cortaría o retrasaría la navegación que existe para explicar.

## Consecuencias

La línea de tiempo de introducción vive en dos lugares que deben coincidir: el `INTRO_TEXT_DELAY_MS` del componente y la duración de la animación CSS `.introIcon`. Las constantes del componente son la fuente de los retrasos de caracteres y del timeout de reconocimiento; el comentario CSS nombra el acoplamiento. El store de asiento gana un bit de estado de UI (`introduce`) que cada stage decide explícitamente, y la sección sigue renderizando un grupo sin miembros — una forma que el golden de la sección y las pruebas unitarias ahora fijan.

## Pruebas

Las pruebas de componente fijan el escalonado limitado (nombre latino de 11 caracteres a pasos de 20ms, nombre CJK de 4 caracteres al tick de 40ms, carácter único sin escalonado), el momento del reconocimiento, y las omisiones de reduced-motion y nombre vacío. `apply.spec.ts` conduce el stage entre pantallas de punta a punta: el borrador de creador se escenifica con la señal activada, un reconocimiento la limpia, y un reconocimiento repetido deja la instantánea intacta. El e2e web `agent-preset-authoring` mantiene el grupo personalizado vacío (encabezado más entrada de creador) en sus goldens.
