# Agent Note: El copy de la fila de comando se divide entre la fila y el handler

Status: implemented

[English](2026-07-30-command-row-copy-contract.md) | [中文](2026-07-30-command-row-copy-contract.zh.md) | Español

## Problema

La fila de comando de la web renderiza `title · summary` a partir de un [par de ciclo de vida de comando](../../proposed/architecture/2026-07-27-session-projection-and-command-log.es.md) registrado: el título era la línea despachada reconstruida a partir de `command/run` (`/permission workspace-write`) y el resumen era el `text` verbatim de `command/done` (`Permission preset: workspace-write.`). Ambas mitades se escribían sin conocer la otra, por lo que la fila decía el nombre del comando dos veces y su argumento dos veces — el peor caso único siendo la fila que un usuario recibe por cada elección del chip Access.

## Decisión

Las dos mitades de la fila tienen trabajos disjuntos, y cada lado se escribe solo para su propia mitad.

El título de la fila es el nombre desnudo del comando — sin `/`, sin argumentos. El `/` pertenece a la gramática de entrada del compositor, no a un registro asentado, y el argumento no es asunto de la fila: el resumen ya dice qué hizo el comando. `GenericCommandCard` conserva el respaldo `命令` para un nodo entre ventanas cuya página `command/run` quedó fuera de la ventana del cliente.

Por tanto, el `text` de cierre de un handler de comando nunca etiqueta su valor con el nombre del propio comando, porque la superficie que lo renderiza ya lo ha dicho. `/permission` devuelve `preset workspace-write`, `current preset workspace-write (available: …)` desnudo, y para un argumento inválido `unknown preset "bogus" (available: …)`. Leída como fila, esto es `permission · preset workspace-write`; leída como línea independiente — la TUI añade el mismo texto como aviso — sigue indicando qué preset se aplica ahora.

La regla prohíbe la *etiqueta*, no el vocabulario. `Permission preset: workspace-write.` perdió porque `Permission preset:` es una leyenda para un valor cuya leyenda ya es el título. Un sustantivo de dominio que casualmente contiene el nombre del comando no es una leyenda y permanece: `/plan` conserva `Plan mode off.` y `Plan mode on. Use /plan off to leave.` (`plan · Plan mode off.` nombra el modo, y la cola es una instrucción, no un eco), y `/goal` conserva `Goal cleared.`. Un handler que se encuentre escribiendo `<Comando> <sustantivo>:` delante de su propio valor es el caso que captura esta regla.

El log no cambia: `command/run` conserva la división estructurada `name`/`args`, de modo que una fila de comando registrado más rica aún puede renderizar argumentos del mismo nodo sin un segundo canal de datos.

## Alternativas consideradas

**Conservar la línea despachada como título y solo acortar el texto de cierre.** El argumento seguiría apareciendo a ambos lados del separador (`permission workspace-write · preset workspace-write`), que es la repetición que se criticaba.

**Eliminar el texto de cierre de la fila colapsada en lugar de los argumentos.** Invierte el valor de la fila: el resultado es para lo que sirve un registro durable, y un texto de error no tendría entonces dónde aterrizar.

**Que la fila elimine un nombre de comando inicial del texto de cierre.** La presentación reescribiría en silencio texto escrito por el handler, y todo handler que redactara su resultado de forma distinta derrotaría la heurística.

**Prohibir por completo el nombre del comando en su texto de cierre, reescribiendo `/plan` y `/goal` para que cumplan.** La prohibición más amplia cuesta más de lo que compra: `Plan mode off.` y `Goal cleared.` son las frases más claras que tienen esos resultados, tanto en la fila como como avisos independientes de la TUI, y los acortamientos que satisfarían una prohibición de nombres (`off.`, `cleared.`) se leen como fragmentos. Las leyendas son la redundancia que vale la pena eliminar.

## Consecuencias

Cada fila de comando se vuelve más corta, y la regla escala: quien escribe un comando nuevo redacta su resultado sin saber qué superficie lo renderiza, y ninguna superficie tiene que deduplicar. El costo es que los argumentos despachados abandonan la fila colapsada — mientras un comando se está ejecutando, la fila muestra solo su nombre y `执行中…` — y que la regla de no leyendas es una convención que hace cumplir el revisor, no una compuerta. Los textos de `/permission` están fijados por las pruebas de comando del paquete permission, y el copy ensamblado de la fila por el golden web [seeded-history](../../../../apps/web/tests/snapshots/seeded-history/command-row.expected.md), que alcanza una fila de comando real asentada sin clave porque `/permission` se ejecuta por completo en el host.
