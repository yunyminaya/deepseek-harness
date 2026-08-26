# Agent Note: Delegaciones de subagentes en paralelo

Status: implemented

[English](2026-08-09-parallel-subagent-delegations.md) | Español

## Problema

Un modelo que quiere hacer fan-out agrupa varias llamadas `subagent` en un único mensaje de asistente — ese lote es la intención paralela. La herramienta de delegación no declaraba ningún clasificador `isConcurrencySafe`, así que el planificador con fallo cerrado ([Agent Note de llamadas de herramienta paralelas](2026-07-10-parallel-tool-call-execution.es.md)) trataba cada delegación en primer plano como una barrera exclusiva: nueve tarjetas en la GUI, un hijo ejecutándose, ocho esperando tras él durante toda su ejecución.

La postura conservadora original — un clasificador unario no puede demostrar que las delegaciones hermanas tengan efectos de workspace disjuntos — había dejado de proteger nada. `run_in_background: true` y las delegaciones continuables ya se solapan con cada llamada posterior, incluidas las escrituras; `dsh-workflow-worker-thread` ya ejecuta hasta su techo de concurrencia de hijos a través de los mismos providers `ctx.subagents.start()` contra el workspace compartido. Solo la variante en primer plano estaba serializada.

## Decisión

`dsh-tool-subagent` declara `isConcurrencySafe: () => true` para toda forma de llamada (primer plano, background de un solo disparo, continuable), así que las delegaciones hermanas de un mismo paso de asistente se solapan bajo el pool rotatorio del loop hasta `maxParallelToolCalls`, con los resultados aún comprometidos en orden de modelo.

La declaración satisface el contrato de seguridad del planificador estructuralmente: un hijo trabaja en su propia sesión, una ejecución nunca muta la sesión del padre (los appends de inicio — `sandbox/mode`, `approval/policy`, `subagent/descriptor` — caen solo en el log del propio hijo), y la herramienta devuelve sus salidas al loop para su commit ordenado. La única escritura propiedad del padre en la forma background de un solo disparo es registrar una Task a través de `tasks.start` — una inserción conmutativa y síncrona que satisface la cláusula de estado compartido de la nota del planificador en lugar de la propiedad más fuerte de no mutación. El seam de providers exige que los inicios concurrentes y las preparaciones continuables de hijos distintos aíslen el estado local de la operación, la cancelación, el asentamiento y la limpieza. Los providers incluidos satisfacen ese contrato: spawn y fork no conservan estado mutable entre inicios, fork solo lee el prefijo de turno completado del padre, los providers fuera de proceso asignan estado por ejecución, y el gestor de continuación reserva una identidad de hijo única y un lock para cada preparación.

Coordinar los efectos de workspace entre hermanos es responsabilidad del modelo, la postura que el producto ya adopta para los hijos de background, continuables y de workflow. Los harnesses pares coinciden: la herramienta Task de Claude Code es incondicionalmente segura para concurrencia (tope 10), la herramienta task de oh-my-pi usa por defecto su clase superpuesta `shared`, la herramienta task de opencode se ejecuta sin límite bajo su SDK, y Codex sortea la cuestión convirtiendo la delegación en un buzón de spawn/espera asíncrono.

La capacidad sigue donde la puso la nota del planificador: `maxParallelToolCalls` limita las llamadas de herramienta sin asentar de un paso — y por tanto los hijos en primer plano en ejecución concurrente — mientras que las llamadas de background y continuables se asientan al inicio y liberan su hueco del pool, así que los hijos que dejan ejecutándose no están limitados por él. Los providers de LLM son dueños de sus propios controles de capacidad.

## Pruebas

Las pruebas de paquete fijan el clasificador para ambas formas de llamada. Una prueba de gate acciona el registro directamente con dos hijos que cada uno se bloquea hasta que ambos hayan iniciado, demostrando la mitad de la que depende la declaración: el cuerpo de la herramienta y la ruta de inicio del provider toleran el despacho concurrente — una serialización oculta en esa pila haría deadlock en lugar de pasar en silencio. Un gate continuable mantiene dos preparaciones de provider en el mismo await, cancela a un llamador antes de la publicación, y demuestra que el hijo cancelado no deja ningún Agent ni Session durable mientras su hermano alcanza la aceptación en la bandeja de entrada y persiste de forma independiente. La mitad de planificación — que la clasificación produzca solapamiento de verdad — es propiedad del pin del clasificador y de la instantánea siguiente.

La instantánea escrita `subagent-parallel` fija el transcript de la aplicación ensamblada: un mensaje de asistente porta dos llamadas subagent, el log del padre registra `tool/call, tool/call, tool/result, tool/result` (la ejecución serial intercalaría pares call/result), y ambos hijos se completan como sesiones separadas. Sus delegaciones gemelas son deliberadamente idénticas: `dsh-llm-replay` vincula los scripts hijo por orden de primera llamada y el recolector ordena los hijos por `createdAt`, y ninguno es determinista entre hijos concurrentes (el marcador `XXX(concurrent-subagents)`), así que solo los gemelos intercambiables se reproducen sin carreras hoy.

## Alternativas consideradas

**Mantener las delegaciones exclusivas.** El statu quo no protegía nada: los hijos de background y workflow ya se solapan libremente con las escrituras, así que serializar la variante en primer plano solo añadía latencia y contradecía la intención explícita de batching del modelo.

**Un clasificador sensible a la entrada.** Los argumentos de la llamada son una descripción en texto libre y un prompt; nada en ellos distingue una delegación segura de una insegura, así que un clasificador condicional sería teatro.

**Un rediseño de spawn/espera asíncrono estilo Codex.** Los hijos continuables más `send_message` ya proporcionan el canal asíncrono; reconstruir el contrato en primer plano en torno a un buzón descartaría una ruta de resultado síncrona que funciona para resolver un problema de planificación que una declaración arregla.

**Un knob de configuración `concurrencySafe` por instancia.** Ningún consumidor necesita un despliegue serial: `maxParallelToolCalls: 1` ya restaura la ejecución serial global, y el arte previo de los harnesses pares fija la delegación como segura para concurrencia por defecto.

## Consecuencias

Los hijos hermanos pueden competir por el workspace compartido o por recursos externos; el modelo es dueño de esa coordinación, como ya lo es para cualquier otro hijo solapado. Los hijos concurrentes también compiten por la cuota del provider de LLM; `maxParallelToolCalls` limita solo las llamadas sin asentar, no los hijos que una llamada de background o continuable dejó ejecutándose.

Dos delegaciones background de un solo disparo en un mismo mensaje adquieren sus ids de trabajo visibles al modelo (`subagent-<n>`) en orden de carrera de despacho. Los ids se registran, así que el replay sigue siendo válido, pero un escenario de instantánea que distinga a sus hijos background heredaría la misma restricción de determinismo que las sesiones hijas gemelas.

Los commits ordenados pueden retener el resultado de un hijo rápido tras un hermano anterior lento — el trade que la [nota del planificador](2026-07-10-parallel-tool-call-execution.es.md) ya aceptó; las superficies vivas siguen mostrando el progreso propio de cada hijo.

Un escenario de instantánea con hijos concurrentes y prompts distintos sigue necesitando soporte del harness de replay (vinculación determinista de scripts hijo y orden de recolección); hasta entonces, esos escenarios deben usar delegaciones gemelas intercambiables.
