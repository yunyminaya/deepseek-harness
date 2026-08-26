# Agent Note: Renderizado de chat de Code Mode — sub-calls como filas nativas bajo el padre

Status: implemented

[English](2026-07-26-code-mode-chat-subcall-rows.md) | Español

> Ámbito: cómo la vista de chat web renderiza un turno `run_code` — la mitad del lado cliente de la pila de UI de Code Mode, construida sobre la [base del host](2026-07-26-code-dispatch-ui-foundation.es.md) (`tool/code-dispatch` de contenido completo, el parámetro `description` obligatorio). La [disolución de toolview](../architecture/2026-07-23-toolview-dissolution.es.md) es dueña del modelo de slots sobre el que esto se apoya.

## Problema

Con Code Mode habilitado, la vista de chat mostraba una única fila `run_code` opaca: texto del programa en bruto como resumen, sub-calls invisibles en todas partes. El requisito de producto asentado es lo contrario: cada sub-call debe renderizarse *idénticamente* a una llamada de herramienta nativa — mismos componentes de fila, mismos registros personalizados, mismo panel de detalles — mientras el transcript sigue siendo honesto sobre el hecho de que el modelo hizo UNA llamada.

## Decisión

**Los sub-calls son bloques estándar de llamada de Tool adjuntos recursivamente a su padre fuera del flujo de superficie, renderizados a través del mismo slot con clave que las filas nativas, y siempre visibles bajo su padre.**

- **Capa de datos**: el `ToolCallTree` del Runtime pliega los eventos `tool/code-dispatch-start` y `tool/code-dispatch` de la ventana en un índice privado por padre, y luego proyecta los hijos en ejecución y asentados sobre los `ToolCallBlock.subCalls` recursivos. La proyección de Live Session y `projectConversationHistory` comparten ese pliegue; los arrays de padres copy-on-write y la proyección de copia de rutas mantienen estables por referencia las raíces y los hermanos no relacionados. Los sub-calls nunca se unen a `nodes` — el flujo de superficie sigue siendo exactamente la estructura de turnos visible para el modelo. Los eventos se estrechan estructuralmente en el límite del consumidor del wire, que también rechaza las relaciones padre cíclicas (los tipos de host de dsh-tools no pueden entrar en el programa cliente porque los merges de `Context` host/cliente colisionan).
- **Capa de render**: `ChatView` pasa cada padre con sus hijos recursivos a través del asiento `'conversation.chat.tool'` de Tool completo. El `ToolCallTree` de ui-tool renderiza el padre seguido de los nidos `[data-subcalls]`, y cada llamada atómica despacha a través del mismo slot con clave `'tool.call.toolview'` con `entryKey = Tool name` y el mismo fallback `GenericToolCard`. Por tanto, un registro con clave se hace cargo de las llamadas descendentes y de nivel superior sin cambios de registro. Los padres en ejecución (`runningCalls`) reciben los despachos acumulados en el mismo bloque recursivo, de modo que las filas hijas llegan en streaming durante la ejecución.
- **Presentación de `run_code`**: una nueva variante de fila `code` (clasificador `run_code → code`, título `Code`, `IconCodeOutline16`) resume con la `description` autorada por el modelo y se expande al propio programa (monoespaciado sobre el relleno de bloque de código markdown) en lugar del sobre JSON de args.
- **Panel de detalles**: `materialFor` busca recursivamente en `nodes` y `runningCalls`, de modo que un callId descendiente seleccionado se resuelve a args completos y salida completa a través de la ruta de renderizado idéntica a la de una llamada asentada nativa.

## Alternativas consideradas

**Sub-calls planos en el flujo de superficie (plegarlos en `nodes`).** Rechazada: tergiversa el transcript — el modelo hizo una llamada; el anidamiento bajo el padre preserva la asociación código↔llamadas y mantiene intacto el invariante de orden visible para el modelo del pliegue.

**Ocultos hasta que la fila del padre se expanda.** Rechazada por decisión de producto: los sub-calls SON la historia de un turno de Code Mode; ocultarlos recrea la opacidad que esta funcionalidad elimina. El conmutador de expansión del padre revela solo el programa.

**Un componente de fila de sub-call dedicado.** Rechazada: el punto central es la identidad con las filas nativas; un componente paralelo derivaría. El envoltorio de nido (sangría + borde izquierdo) es el único chrome específico de sub-call.

## Consecuencias

Los registros de toolview personalizados se aplican a los sub-calls de forma gratuita — y deliberadamente: no hay exclusión voluntaria por registro salvo que el componente lea su propio contexto, algo que ningún consumidor actual necesita. El resaltado de selección llega a las filas anidadas a través del mismo canal `selectedCallId` (la pertenencia a grupos busca en todo el árbol). Trajectory/waterfall ahora dibujan los tramos de sub-call a partir del par de timing de despacho ([live parallel dispatch](2026-07-26-code-mode-live-parallel-dispatch.es.md)); sin ese timing, un tramo de waterfall sería una mentira. El turno 64 del fixture (`?fixture`) más el e2e de navegador `code-mode-round` (ronda real grabada, replay sin clave) fijan la superficie completa; las suites de jsdom y Runtime fijan el dispatch de slots, los estados de error, la resolución de detalles recursiva, la proyección de historial y la copia de rutas estable por referencia.
