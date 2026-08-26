# Ejemplos de estilo de traducción (style samples)

[English](style-samples.md) | Español

Este archivo es el ancla de calibración del estilo de traducción: cada grupo de ejemplos empareja un original en inglés con una traducción al español terminada a mano, y cubre los principales registros de la documentación de este repositorio. **El estilo de la traducción se rige por estos ejemplos** — los ejemplos de estilo prevalecen sobre las descripciones verbales del tono, pero la tabla de terminología, la fidelidad y las reglas de estructura siguen teniendo prioridad. Al traducir o revisar, contrástalo con el ejemplo de estilo más cercano. Este archivo es bilingüe por construcción (inglés–español) y no participa en el emparejamiento (véase la lista de exclusiones en [README.es.md](README.es.md)).

Mantenimiento: cuando la revisión humana calibre nuevos pasajes de referencia, se añaden al registro de estilo correspondiente; los errores de semántica, estructura o terminología se corrigen directamente. Todo ejemplo nuevo o corregido pasa por una revisión de PR.

## ① Narración de arquitectura

> This document describes the architecture of the DeepSeek Harness — the foundation of **DeepSeek Code**. The governing principle, from the microkernel design discussion: **everything is a plugin**. The core is deliberately tiny — a handful of abstract services plus one concrete loop plugin (`dsh-agent-loop`) — and every product feature is a plugin against the extension API described here, without modifying the loop.

Este documento presenta la arquitectura general de DeepSeek Harness, la base subyacente de **DeepSeek Code**. La discusión de diseño del microkernel fijó el principio rector: **todo es un plugin**. El núcleo es deliberadamente mínimo: solo un puñado de servicios abstractos y un plugin de bucle concreto, `dsh-agent-loop`. Todas las funcionalidades del producto se desarrollan como plugins independientes sobre la interfaz de extensión definida en este documento, sin modificar la lógica del bucle principal.

> Dependency rule: extension plugins depend on interfaces, never on `dsh-agent-loop` (the loop is swappable); the sanctioned exception is the composition bundle `dsh-agent-spine-demo`, whose job is assembling the concrete spine.

Regla de dependencias: los plugins de extensión dependen únicamente de interfaces abstractas y tienen terminantemente prohibido depender directamente de `dsh-agent-loop` (ese bucle principal admite implementaciones de reemplazo); la única excepción permitida es el paquete de composición `dsh-agent-spine-demo`, cuya responsabilidad es ensamblar la columna vertebral concreta.

> This document covers **behavior**; type definitions live in [subsystems/](../subsystems/core.md), the per-event/service reference lives in the generated regions of [subsystems/](../subsystems/core.md), and package contracts in the package READMEs state each package's required configuration and behavior ([map](../../packages/README.md)).

Este documento describe el comportamiento general; las definiciones de tipos viven en [subsystems/](../subsystems/core.es.md); la referencia detallada de cada evento y servicio está en las regiones generadas de [subsystems/](../subsystems/core.es.md); y los README correspondientes exponen la configuración y el comportamiento que exige cada paquete ([índice](../../packages/README.es.md)).

## ② Reglas de patrones defensivos

> Hard-won bug-class rules: each pattern below is a class of defect that actually shipped or nearly shipped here, stated as the rule that prevents its recurrence. Read this before writing lifecycle, concurrency, subprocess, or teardown code.

Reglas de clases de defectos ganadas a pulso: cada patrón de abajo corresponde a una clase de defecto que aquí llegó a producción, o estuvo a punto de llegar, y se formula como la regla que impide que vuelva a ocurrir. Antes de escribir código de ciclo de vida, concurrencia, subprocesos o destrucción de recursos, lee sin falta este documento.

> **Dispose must reach quiescence, not just request it** — A teardown that issues kills/aborts but returns before the work stops leaves orphans. Make cleanup async and await the children's exit (kill → await `done`), and close listener/notification registries BEFORE killing so late completions stay silent. Tests prove disposal waited (pid gone right after `await fiber.dispose()`), not merely that the process eventually dies.

**dispose (liberación de recursos) debe esperar a que todas las tareas queden completamente en reposo; no basta con emitir la orden de terminación y regresar**: si el proceso de limpieza solo emite señales de terminación o interrupción y regresa sin esperar a que las tareas se detengan, quedan procesos huérfanos. La limpieza debe ser asíncrona: esperar a que todas las subtareas salgan por completo (primero emitir la señal de terminación y luego esperar la salida); antes de emitir la señal, cierra los registros de listeners y notificaciones, para que los eventos de finalización que lleguen tarde no disparen notificaciones. Las pruebas deben demostrar que dispose esperó de verdad a que la limpieza terminara: el PID desaparece inmediatamente después de ejecutar `await fiber.dispose()`; no basta con comprobar que el proceso acaba muriendo por sí solo.

> **Async state is not synchronous state** — `agent.followup()` does not flip status before returning; a background job's completion races turn boundaries; `reader.close()` fires for both EOF and disposal. Never gate control flow on a status you only just requested — drive lifecycle off the events/promises that actually fire (`agent/status`, `task.done`), and observe the transition (saw `running` THEN `idle`) instead of treating status as a per-follow-up result: several queued follow-ups run as consecutive turns under one `running` interval, while cancellation or disposal can discard unstarted items.

**El estado asíncrono no es el estado síncrono instantáneo**: llamar a `agent.followup()` no actualiza el estado de forma síncrona antes de regresar; el momento en que termina una tarea de segundo plano entra en carrera con los límites de turno; `reader.close()` se dispara tanto al llegar al final del archivo como al liberar recursos. Nunca condiciones el flujo de control a un estado que acabas de solicitar; la lógica del ciclo de vida debe regirse por los eventos y las promises que se disparan de verdad (`agent/status`, `task.done`), y observar la transición completa (primero `running`, luego `idle`), sin tratar el estado como el resultado de cada `followup()`: varios `followup()` encolados se ejecutan como turnos consecutivos, pero pueden compartir el mismo intervalo `running`; la cancelación o la liberación de recursos también pueden descartar elementos de la cola que aún no han arrancado.

## ③ Lista de políticas de pruebas

> **Coverage gate** (`pnpm run test:coverage`): the gating run, per-file 100% on `packages/*/*/src`. An uncovered line is often dead code the gate is correctly flagging for deletion, not a missing test to bolt on. Line coverage is necessary, never sufficient — it proves lines ran, not that the feature works as shipped.

**Compuerta de cobertura** (`pnpm run test:coverage`): como comprobación de la compuerta de fusión, exige un 100 % de cobertura de líneas por archivo en `packages/*/*/src`. Las líneas sin cubrir suelen ser código muerto inútil que la compuerta señala para su eliminación, no una prueba que falte por añadir. La cobertura de líneas es una condición necesaria, pero en absoluto suficiente: solo demuestra que el código se ejecutó; no garantiza que la funcionalidad cumpla lo esperado en producción.

> We are DeepSeek — do not ration real-API tests. A no-key test proves the plumbing; only a with-key run proves the agent works against a real model. Write many: real prompts that write files, multi-turn conversations, tool use, cancellation mid-stream. Cheapest and highest-value are **smoke tests** that boot the real example, send one real prompt, and check the world — they catch the "green unit tests, broken product" class that mocks structurally cannot. The self-skip exists only so secretless CI and keyless contributors aren't blocked; it is not a cost signal.

Somos DeepSeek: las pruebas de API real no deben recortar deliberadamente el número de casos. Una prueba sin clave solo verifica el conducto subyacente; solo una ejecución con clave válida confirma que el agent (agente) se conecta correctamente a un modelo real. Escribe muchas: prompts reales que escriben archivos, conversaciones de varios turnos, llamadas de herramienta, cancelación a mitad del stream.

Las más baratas y las que más valor aportan son las **pruebas de humo (smoke tests)**: arrancan el ejemplo real completo, envían un prompt real y comprueban los resultados externamente observables (archivos, procesos, etc.). Estas pruebas capturan una clase de problemas —las pruebas unitarias en verde con el producto roto en ejecución— que los mocks, por sí solos, jamás podrían detectar.

La lógica de auto-omisión incorporada existe únicamente para que el CI sin secretos y los contribuyentes sin clave no queden bloqueados por el proceso; no es una señal de coste ni una razón para recortar la inversión en pruebas de API real.

> **Prefer the real implementation over a mock** — Mock only genuinely expensive or non-deterministic dependencies (the LLM adapter, the network, the clock); keep everything downstream real. A hand-rolled stand-in proves the bridge moves bytes, not that the shipping tool behaves as asserted — the two drift while the test stays green.

**Prefiere la implementación real a un mock** — Haz mock solo de las dependencias realmente caras o no deterministas (el adaptador del LLM, la red, el reloj); todo el resto, río abajo, usa la implementación real. Un sustituto hecho a mano solo demuestra que el puente mueve bytes, no que la herramienta publicada se comporta como se afirma; con el tiempo, la lógica real y el mock divergen, pero la prueba sigue en verde.

## ④ Descripción del mecanismo

> Blob hashes, not commit hashes, so the record is computable for files edited in the same PR (`git hash-object foo.md`) and consistency is a pure content comparison. The recorded hash also recovers the exact last-confirmed text of either side (`git cat-file -p <hash>`), so an out-of-sync pair is updated by diffing the edited side against its last-confirmed state and patching the counterpart minimally — never by re-translating whole files.

El sistema registra el estado mediante el blob hash del archivo, no el commit hash. Al modificar archivos dentro del mismo PR, el blob hash correspondiente se calcula directamente con `git hash-object foo.md`, y basta con comparar el contenido de los archivos para saber si el par bilingüe está sincronizado. Con el blob hash registrado, `git cat-file -p <hash>` permite recuperar el texto exacto que ambos lados tenían en la última confirmación de alineación. Cuando el par bilingüe queda desincronizado, solo hay que comparar la versión modificada con la última versión confirmada y aplicar al otro lado el cambio mínimo de sincronización, sin volver a traducir archivos enteros.

## ⑤ Declaración de política

> The gate's limit, stated plainly: a green gate means the pair was confirmed consistent at these exact contents, not that the confirmation was sound. It checks hashes and Markdown structure; it cannot judge whether the two sides actually say the same thing — that is the reviewer's half of the contract. A re-recorded pair with a sloppy counterpart passes the gate; it must not pass review.

La limitación de la compuerta es clara: pasar la compuerta solo indica que el blob hash actual de ambos archivos coincide con el registro acompañante y que la firma estructural de Markdown es coherente, es decir, que este contenido fue confirmado como coherente en algún momento; no significa que esa confirmación sea fiable. El revisor debe comprobar que las dos lenguas expresan de verdad lo mismo. Aunque la traducción sea tosca y el significado esté mal, tras volver a registrar el par se sigue pasando la compuerta; jamás debe pasar la revisión humana.

## ⑥ Argumentación de Agent Note

> Comparing git timestamps of the pair (no record) — rejected: formatting-only edits would false-positive, and a counterpart committed after an unrelated edit would false-negative; content identity is the only signal that means what the gate claims.

Comparar las marcas de tiempo de git del par (esquema sin registro) — se rechaza: los cambios que solo ajustan el formato dispararían falsos positivos, y confirmar la traducción después de una modificación no relacionada produciría falsos negativos. Solo la identidad basada en el propio contenido (la comparación del blob hash de cada lado con el registro acompañante) puede sustentar la semántica que la compuerta afirma.

## ⑦ Requisito universal (demostración de división de párrafos largos)

> **Universal requirement**: every in-scope document merges as a complete bilingual pair. The manifest contains only explicit exclusions: it has no per-file rollout list, date cutoff, or README-specific policy class. […] Pairing is a continuing obligation: every later edit to either side updates the counterpart and consistency record in the same change.

**Requisito universal**: todo documento dentro del alcance debe fusionarse como un par bilingüe completo. El manifest (manifiesto) solo contiene exclusiones explícitas: no hay lista de implementación por archivo, ni fecha límite, ni clase de política específica de README. (…) El emparejamiento es una obligación continua: cuando se modifica cualquiera de los dos lados, el mismo cambio debe actualizar la contraparte y el registro de consistencia.

## Puntos clave extraídos de los ejemplos

- El estilo es el del texto normativo: oraciones con sujeto y predicado completos y tono asertivo; nada de registro coloquial ni de tono académico.
- Da a las oraciones un sujeto ejecutor explícito: las pasivas del inglés y los sujetos abstractos se escriben en español con «el sistema / la compuerta / la herramienta / el revisor» como sujeto.
- Sustituye la traducción literal por la jerga consolidada de la ingeniería en español: false positive/negative → falso positivo / falso negativo; ratchet → solo se aprieta hacia delante, nunca se afloja; reviewable act → acto revisable.
- Localiza las metáforas en lugar de transplantarlas: bilingual from birth → bilingüe desde su creación; grandfathered → legado histórico.
- Los nombres de categoría se dicen en español y en la primera aparición se anota el inglés entre paréntesis: recetario (cookbook), post-mortem (postmortem); cuando se refieren a un directorio o una ruta, conservan el inglés con formato de código.
- Divide los párrafos largos por unidad semántica, una idea por párrafo; expande las frases nominales en oraciones verbales.
- Reescribir en la lengua materna no es recortar: cada componente semántico del original tiene que materializarse.
- Cuando un ejemplo entre en conflicto con [terminology.es.md](terminology.es.md), manda la tabla de terminología: antes de incorporar un ejemplo, corrige los términos según la tabla (por ejemplo, agent, mock y LLM se conservan en inglés; cancellation se traduce «cancelación»).
- Los identificadores con formato de código (nombres de eventos `agent/status`, valores de estado `running`, nombres de paquete `dsh-bash-local`, etc.) conservan en la traducción su code span original; no deben reformularse de manera coloquial; el Pass 2 debe verificar cada frase.
