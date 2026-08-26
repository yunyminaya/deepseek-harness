# Agent Note: Gracia de puntero para popups de hover

Status: implemented

English | [中文](2026-07-30-hover-popup-pointer-grace.zh.md)

## Problema

Ambos popups que levantan las filas del navegador de workspace flotaban fuera del alcance del puntero. `HoverCard` cerraba en el primer `pointerleave` de su ancla y renderizaba su tarjeta con `pointer-events: none`, pero la tarjeta se sienta 8px fuera del borde derecho del ancla, así que todo camino hacia ella cruzaba terreno de ninguno y mataba la tarjeta antes de llegar — la ruta completa de workspace y el título de sesión que existe para mostrar solo podían leerse al paso. Los menús de acción de fila pasaban `closeOnPointerLeave`, cuyo handler vivía en la lista portaled: apuntar de vuelta al disparador `...` que abrió la lista la cerraba, y lo mismo cualquier paso en largo más allá de un borde de lista, sin ventana para volver.

## Decisión

`usePointerGrace` ([packages/client/ui-primitives/src/pointer-grace.ts](../../../../packages/client/ui-primitives/src/pointer-grace.ts)) posee un cierre demorado cancelable, compartido por ambos átomos, con `POINTER_GRACE_MS` en 200. Salir arma el cierre; volver lo cancela. El tránsito por un hueco ancla→popup es por tanto sobrevivible, mientras que un puntero que genuinamente se ha ido sigue descartando el popup.

`HoverCard` arma la gracia al salir en vez de cerrar, y su tarjeta ya no pone `pointer-events: none`, así que descansar sobre la tarjeta la mantiene abierta. Re-entrar estando ya abierta cancela el cierre pendiente sin reiniciar la espera, lo que evita que la tarjeta parpadee cuando el puntero cruza el hueco. Una presión sobre la tarjeta inicia una selección en vez de descartarla; solo las presiones en la región del ancla y un propietario que ponga `disabled` descartan de inmediato, por delante de la gracia.

`Menu` mueve el descarte por salida de puntero de la lista portaled al span envolvente. El recorrido enter/leave de React corre sobre el árbol React, así que el disparador y la lista portaled son allí una región: cruzar el hueco de 4px entre ellos, o apuntar de vuelta al disparador, ya no cuenta como salir. La salida solo se arma mientras la lista está abierta, y un cierre impulsado por el propietario (selección, Escape, clic fuera) desarma un cierre de gracia pendiente en un efecto keyed solo en `open` — plegarlo en el efecto de clic-fuera cancelaría la gracia en cada re-render, ya que los propietarios pasan un closure `onClose` fresco cada vez.

## Alternativas consideradas

**Cerrar los popups solo con clic fuera y Escape.** Descartado porque ambos popups se levantan por hover y van sin etiqueta de descartables; dejarlos abiertos tras mover el puntero a otra fila dejaría una tarjeta varada sobre contenido no relacionado.

**Ensanchar el área de hit del ancla hasta tocar el popup.** Descartado porque los offsets de 8px y 4px son del diseño, y un elemento puente invisible tendría que seguir cada reposicionamiento que los popups de posición fija ya hacen en scroll y resize.

**Conservar la tarjeta de hover con `pointer-events: none` y solo añadir la gracia.** Descartado porque el puntero descansando sobre la tarjeta golpearía lo que haya detrás, así que la gracia expiraría y cerraría la tarjeta que el usuario acababa de alcanzar.

**Dar a cada átomo su propio timer.** Descartado porque los dos cierres son el mismo comportamiento con la misma afinación; un hook compartido evita que se desvíen entre sí.

## Consecuencias

La tarjeta de hover ahora es hit-testable y cubre 244px de lo que superponga mientras se muestra, que es el precio de ser alcanzable; sigue viva solo mientras el puntero esté en la fila o la tarjeta. Los menús de fila sobreviven el round trip entre disparador y lista, y un menú que cierra por su propia razón no puede reabrirse hacia un cierre pendiente stale. Los menús sin `closeOnPointerLeave` quedan intactos — los handlers del envolvente solo se conectan cuando está puesto.

## Testing

`packages/client/ui-primitives/tests/hover-card.client.spec.tsx` y `tests/atoms.spec.tsx` fijan el límite de la gracia, la cancelación-al-volver, la no-segunda-espera, el desarme-al-cierre-del-propietario y el caso de no-armarse-cerrado. Los gestos de alcanzabilidad en sí — posarse sobre la tarjeta, y moverse entre una lista abierta y su disparador — quedan fijados en el browser real por `apps/web/tests/workspace-management.e2e.ts`, ya que dependen de hit testing y layout que jsdom no modela.
