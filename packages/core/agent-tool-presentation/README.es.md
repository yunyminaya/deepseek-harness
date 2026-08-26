# dsh-agent-tool-presentation

[English](README.md) | Español

La fila que un [agent preset](../../preset/agent-presets/README.es.md) lleva para indicar qué forma de sus herramientas ve el modelo: `native` (todos los schemas), `code` (solo `run_code` más un SDK de TypeScript generado) o `both`.

## Por qué una fila y no un registro

El registro de herramientas no puede moverse a un preset. Sus Consumers están todos en el plano de host — [`dsh-agent-loop`](../agent-loop/README.es.md) lee su planificador, [`dsh-apiproxy`](../../host/apiproxy/README.es.md) lee sus presenters para renderizar las tarjetas de herramientas, y cada plugin de herramientas se registra en él — y un servicio solo desciende cuando todos sus Consumers descienden con él.

Lo que un preset sí puede poseer es la **presentación** de ese registro. `ctx.tools.presentAs()` la declara solo para el agent que la monta, así que una sesión en Code Mode convive con las nativas en un mismo proceso, viendo cada una su propio catálogo. El `mode` del despliegue en la fila de [`dsh-tools`](../tools/README.es.md) sigue siendo el valor por defecto que reciben los agents que no declaran nada.

## Qué hace

`native` se aplica de inmediato. Un code mode, en cambio, espera a `ctx.codeRuntime`, que es un servicio del plano de host ([`dsh-code-runtime-worker-thread`](../../code-runtime/code-runtime-worker-thread/README.es.md)): un preset que selecciona Code Mode contra un despliegue que no compone ningún runtime mantiene entonces esta fila pendiente, y `dsh-agent-presets` rechaza el montaje que nombra este id. La alternativa — aplicar de forma optimista — traslada el fallo a la primera solicitud de la sesión, donde el operador no puede actuar ni sobre el preset ni sobre la composición.

`mode` es obligatorio y no tiene valor por defecto, porque un preset sin esta fila ya recibe el valor por defecto del despliegue; un valor omitido significaría que la fila se compuso para nada.

Un agent declara una sola presentación. Una segunda declaración en la misma composición se rechaza en lugar de fusionarse: dos respuestas a «qué forma ve el modelo» es una contradicción, no una anulación.

## Experiencia del modelo

Indirectamente, a través de la proyección que selecciona en `dsh-tools`: `code` presenta `run_code` más una sección de SDK generada y la regla de que solo `run_code` puede llamarse directamente, `native` presenta el schema de cada herramienta. La selección decide también qué puede EJECUTARSE: bajo `code`, el registro resuelve una llamada directa al modelo que nombre cualquier otra herramienta a `UNKNOWN_TOOL`, así que esta fila es lo que mantiene la superficie anunciada y la superficie invocable iguales para cada agent al que cubre ([nota sobre el colapso del ejecutor](../../../.agents/notes/implemented/bug-fix/2026-08-07-code-mode-executor-collapse.md)).

#### Efecto de KV Cache

Sin invalidación directa; la presentación se fija cuando se compone el agent, así que su prefijo de solicitud es estable durante toda la vida de la sesión.

## Limitaciones conocidas y trabajo diferido

- **El runtime sigue en el plano de host** — un preset puede seleccionar Code Mode pero no puede suministrar el runtime de TypeScript que necesita; un despliegue que no compone ninguno no puede componer ningún preset de code mode.
