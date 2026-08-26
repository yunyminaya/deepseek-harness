# Agent Note: Qué permanece en el plano de host cuando los preajustes poseen el plano del agent

Status: implemented

[English](2026-08-10-host-plane-ownership-after-presets.md) | Español

## Problema

Los [preajustes de agent por sesión](2026-08-03-per-session-agent-presets.es.md) movieron toda fila orientada al modelo al plano del agent, y cada arreglo posterior ha sido un lector que asumía el mundo anterior al movimiento. `tasks` volvió al host porque una fila de preajuste fuera de su realm la resolvía; `goals` nunca se fue por la misma razón; el `toolFilter` de un agent hijo se reparó una vez que toda herramienta orientada al modelo se convirtió en contribución de ancestro en lugar de global ([los agent hijos se unen al preajuste de su padre](../bug-fix/2026-08-10-child-agents-join-their-parent-preset.es.md)).

Dos lectores más seguían del lado equivocado de esa línea.

`dsh-token-meter` estaba deshabilitado en el host y montado dentro del realm `compaction` de cada preajuste. No toma configuración, indexa cada pliegue por `Session` y no registra herramienta ni sección de prompt — pero posee las unidades de proyección `tokenUsage`, `contextPressure` y `contextBreakdown`, y `sessionProjections` es una tabla a nivel de proceso sin capas por ámbito. Una unidad registrada desde dentro de un preajuste responde por tanto por todas las sesiones: que una sesión `minimal` mostrara un medidor de contexto dependía de que alguna *otra* sesión hubiera montado `standard` desde el arranque, y un proceso que solo hubiera ejecutado `minimal` no mostraba ninguno.

Nada nombraba a un agent que no se uniera a ningún preajuste. La unión es un enlace de padre de ámbito; sin ella, las vistas `tools`, `system-prompt` y `skill` resuelven la capa global vacía y el modelo no recibe nada — sin error, sin catálogo vacío, solo un agent que no puede actuar. Así es como los subagentes delegados se ejecutaron mientras existieron los preajustes, y el mismo hueco está abierto en todo punto de entrada anterior a ellos.

## Decisión

**El medidor es del plano de host.** `dsh-token-meter` vuelve a la composición de host y abandona el mapa `isolate` de los preajustes, de modo que `compaction-basic` y `tool-result-pruner` resuelven la única instancia de host desde dentro de su realm. Los preajustes conservan el realm y el backend — lo que elige un preajuste es si su agent compacta, no si sus tokens se cuentan. Este es el criterio por el que ya se leen `tasks` y `goals`, aplicado a un servicio cuyo alcance de *proyección* es lo que hacía errónea la propiedad del preajuste: una unidad cuyo valor vacío es indistinguible de uno real no puede ser por-composición mientras la tabla en la que se registra sea por-proceso.

**Un agent no unido se nombra dos veces, en dos puntos distintos.** `AgentPresets` registra un aviso por cada agent publicado con una cadena de ámbito de longitud uno mientras hay una plantilla configurada. El acompañante de invariante falla en su lugar —y en `system-prompt/assemble`, no en la publicación, porque un agent no unido es legal hasta que se dirige a un modelo: `recompose` enlaza exactamente a un agent así como primer eslabón, y el ensamblaje de prompt es el único llamante que aporta un ámbito de agent, así que un ensamblaje de host y un montaje permanente quedan ambos correctamente fuera de alcance.

Tres límites siguen abiertos y se registran donde muerden en lugar de arreglarse aquí: la presencia de clave de proyección no es una señal de capacidad por sesión ([`dsh-session-projection`](../../../../packages/session/session-projection/README.md)); una generación permanente superada nunca se reclama, lo que el flujo de autoría de la página de ajustes convierte en un coste por guardado ([`dsh-agent-presets`](../../../../packages/preset/agent-presets/README.md)); y un plugin temporal montado a través de `cordis_mount` pertenece a la composición y no a la sesión que lo montó ([`dsh-tool-cordis`](../../../../packages/extensions/tool-cordis/README.md)).

## Pruebas

`apps/cli/tests/web-agent-presets.e2e.ts` lee `ctx.get('tokenMeter')` en la composición Web arrancada antes de que ningún preajuste del archivo se monte —un medidor del lado del preajuste está tras un realm `isolate` y es invisible para `ctx.get`, así que la lectura es una aserción de propiedad y no una coincidencia de orden de montaje— y luego afirma que la instantánea de una sesión `minimal` lleva las tres unidades.

`packages/preset/agent-presets/tests/mount.spec.ts` afirma que el aviso salta exactamente una vez para un agent desnudo y nada para uno unido. `tests/invariant.spec.ts` lleva el control negativo: el ensamblaje de un agent no unido se rechaza, mientras que el ensamblaje de un agent unido y un ensamblaje de host sin ámbito pasan ambos.

## Alternativas consideradas

**Conservar el medidor en el preajuste y dar capas por ámbito al registro de proyección.** El arreglo preciso, y mucho mayor: `snapshot`, `checkpoint` y la conducción ansiosa necesitarían cada una una resolución sesión→ámbito que una lectura fría no tiene sin el `presenterScopeFor` de api-proxy. Rechazado como desproporcionado para un solo servicio sin estado por preajuste alguno; la regla general se documenta en el registro en su lugar.

**Vetar la publicación de un agent no unido.** Lo ruidoso gana a lo silencioso, y el registro lo soporta — un listener síncrono de `agent/created` que lanza revierte la creación. Rechazado porque componer un agent fuera de la plantilla es legal: `recompose` documenta el agent desnudo que luego enlaza, y el puente ACP, el servidor SDK y el bundle headless crean uno hoy. Un veto convertiría una brecha de capacidad en una caída.

**Comprobar la unión en `agent/created` también en el acompañante.** Rechazado: la publicación no puede distinguir una unión perdida de un agent que se enlazará después, así que la comprobación rechazaría una vía documentada. El ensamblaje de prompt puede distinguirlos.

**Mover `plan-mode` y `tool-todo` fuera del plano del agent por la misma razón de proyección.** Rechazado: ambos son capacidades genuinamente por preajuste, y sus unidades calculan un valor vacío para una sesión que nunca las usa, que los clientes ya leen por valor (`plan.active`, una lista vacía). Solo una unidad cuyo valor vacío es indistinguible de uno real —el medidor— obliga a la propiedad del host.

## Consecuencias

El medidor de contexto se convierte en un hecho por sesión en lugar de una función del historial de montajes. Un preajuste ya no puede optar por no participar en la contabilidad de tokens; ningún preajuste enviado lo hacía, y `minimal` ahora dice que descarta la auto-compactación en lugar de la contabilidad.

El aviso es consultivo, así que un despliegue que añada una plantilla a los puntos de entrada ACP o servidor SDK sigue arrancando agents sin herramientas — solo que lo dice una vez por agent en lugar de en silencio. El invariante solo alcanza a las composiciones que cargan `dsh-invariants`, que cerca los tests de paquete y los hosts de desarrollo, no a una de producción.
