# Agent Note: El registro de skills es del host y se organiza en capas por ámbito

Status: implemented

[English](2026-08-09-layered-skill-registry.md) | Español

## Problema

La pila de preajustes de agent movió toda la capacidad de skill —registro, provider local y la herramienta `skill`— al realm `isolate` de cada preajuste, porque «qué skills tiene un agent» es una elección del plano del agent. Ese encuadre confundía dos preguntas distintas: qué skills aporta un *despliegue*, y si un *agent* las consume. El wrapper preparado de un plugin de repositorio declara `inject: ['skills']` y monta su raíz de skills como provider del plano de host; sin registro de host compuesto en los perfiles web y headless, ese wrapper esperaba para siempre y el e2e del plugin de repositorio se colgaba, lo que en su momento se evitó eliminando la raíz de skills del fixture. Un registro por realm de preajuste además hacía que el listado de skills del gateway dependiera de un agent vivo — el popup `/` de una sesión fría no tenía registro alguno que leer.

El registro de herramientas nunca tuvo este problema: es un singleton de host con capas por ámbito sobre `dsh-scope`, así que las herramientas a nivel de despliegue (servidores MCP, entradas de plugin) se registran globalmente mientras las filas de un preajuste se registran en la capa de ese preajuste.

## Decisión

`SkillRegistry` adopta la misma forma. Contiene `ScopedLayers<SkillLayer>`; `registerProvider()` y `register()` archivan en la capa del ámbito del contexto llamante, de modo que las filas de host y los plugins de repositorio aterrizan en la capa global mientras que el `skill-filesystem` de un preajuste —montado por la composición permanente, cuyo contexto lleva la clave de ámbito del preajuste— aterriza en la capa de ese preajuste. Los nombres de provider son únicos por capa y no a nivel de proceso, que es lo que permite a cada preajuste montar su propio provider `local`.

Las lecturas toman el ámbito de visualización a través de `SkillViewOptions` (el agent llamante, que es su propia clave de ámbito). El registro fusiona la capa global con la cadena del ámbito: **la capa más cercana gana un nombre duplicado de plano, y el rango decide los duplicados solo dentro de una capa** — la regla de sombreado del registro de herramientas. Se consideró y rechazó agrupar rangos entre capas: los rangos se diseñaron para ordenar fuentes que se conocen entre sí, y bajo un grupo global un plugin de repositorio instalado después podría desplazar silenciosamente la skill homónima propia de un preajuste mediante desempate por orden de registro, cambiando el comportamiento de una composición a distancia. El que gana el más cercano mantiene el comportamiento de una composición decidido por su autor.

Los cachés de detección se indexan por la cadena de ámbito resuelta más un contador de revisión, de modo que una recomposición de sesión en blanco —que re-padrea la clave de ámbito del agent sin tocar el registro— sea visible para la siguiente lectura.

La composición se mueve con ello: el bundle web-app re-habilita la fila base del registro `skill` (solo `skill-filesystem` y `tool-skill` siguen siendo propiedad del preajuste), y las composiciones de preajuste dejan caer su realm `isolate: skills` por filas desnudas sobre el registro de host. El dominio de skills del gateway lee el registro de host en el ámbito del presentador —el agent vivo, si no la clave permanente registrada del preajuste— de modo que una sesión fría lista el catálogo que su composición sirve de verdad en lugar de fallar; la rama `serviceFor` se mantiene para composiciones que aún montan su propio registro por realm.

## Consecuencias

**Una skill a nivel de despliegue llega a toda sesión compuesta por preajuste que monte `tool-skill`.** La raíz de skills del e2e del plugin de repositorio y sus aserciones se restauran; el e2e de Web enviada demuestra que la fila de insignia (la misma forma de registro de host) se fusiona en el catálogo de un agent de preajuste estándar mientras la vista de host sigue siendo solo-global.

**La visibilidad de capas y el consumo siguen siendo elecciones separadas.** Un agent `minimal` puede leer la capa global en principio, pero no compone ninguna herramienta `skill` — si un agent tiene skills o no sigue siendo decisión del preajuste, tomada al montar u omitir `tool-skill`.

**Las opciones de provider siguen siendo el objeto llamante prestado.** `SkillViewOptions` extiende `SkillLookupOptions`; el registro consume `scope` y los providers leen solo su propio contrato del mismo objeto de solo lectura, conservando la garantía existente de identidad prestada.

**El perfil TUI no se ve afectado.** Con todas las filas en host, hay exactamente una capa (global) y la vista fusionada equivale a la antigua vista de registro único, rangos incluidos.

**El sombreado entre capas es silencioso.** Dentro de una capa, el perdedor se registra como antes; una capa más cercana que reemplaza un nombre más lejano sigue la convención del registro de herramientas y no registra nada. El registro sigue sin exponer API para inspeccionar definiciones sombreadas.

## Alternativas consideradas

**Grupo de rangos en todas las capas visibles.** Fiel a la precedencia de registro único, pero los empates entre capas se rompen por orden de registro (los providers de arranque ganan siempre a los montajes permanentes), y la skill propia de un preajuste podría ser desplazada por un cambio de despliegue que nunca ve. Rechazado por estabilidad de composición; véase Decisión.

**Conservar los registros de realm por preajuste y entregar las skills de repositorio como directorios que escanea el provider de un preajuste.** Deja roto el contrato `inject: ['skills']` del wrapper (o bifurca el wrapper por perfil), duplica la configuración de detección en cada preajuste y aun así no da nada que leer a las sesiones frías. Rechazado.
