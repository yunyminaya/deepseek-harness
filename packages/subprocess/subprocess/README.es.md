# @deepseek-ai/dsh-subprocess

[English](README.md) | Español

El seam de subprocess (`ctx.subprocess`) es la mitad de procesos de un único mundo de ejecución. El `SubprocessRuntime` abstracto expone la búsqueda de ejecutables, el `spawn` gestionado ordinario y un único primitivo de proceso de terminal; su vocabulario cubre stdio crudo/recogido, manejadores de proceso y terminal, datos de salida (exit facts), limpieza de árbol/sesión y el espacio de nombres de entorno gestionado `DSH_*`. La implementación local vive en [`dsh-subprocess-local`](../subprocess-local/README.es.md).

## Contrato

- `spawn(spec)` devuelve de inmediato un manejador en vivo; `done` se resuelve al cerrarse el proceso con los datos de salida (`SubprocessOutcome` no lleva salida ni clasificación de causa) y solo se rechaza por fallos a nivel de spawn.
- Los directorios de trabajo de spawn y las rutas de los ejecutables pertenecen al mundo de ejecución del provider. `resolveExecutable(command, env?, signal?)` verifica los comandos absolutos o resuelve los nombres simples contra el PATH depurado de ese mundo más las anulaciones explícitas.
- El spec es completamente explícito — argv, cwd, disposiciones de stdio por flujo, gracia — porque los valores por defecto que varían según el despliegue pertenecen a la configuración del llamador, no a un valor por defecto oculto del servicio de subprocess (la división solicitud/spec de `dsh-shell` es la plantilla propietaria). `argv` nunca se interpreta como shell; un Consumer que quiera un shell pasa él mismo `['bash', '-c', command]`.
- El stdio tiene forma de Node por flujo: `'pipe'` entrega al llamador el flujo crudo para su propio encuadre de protocolo (LSP JSON-RPC, ACP ndjson), `'inherit'` pasa el descriptor del padre para diagnósticos, y el modo de recogida (`{ maxBytes, spill? }`) almacena en búfer una cola acotada con un archivo de spill opcional de flujo completo. Los lectores de recogida toman offsets de bytes de flujo completo y nunca consumen, así que lectores independientes no pueden robarse los deltas entre sí; una lectura cuyo offset se ha salido de la cola en memoria es `lossy` y apunta al archivo de spill cuando existe. La salida recogida sigue siendo legible tras el asentamiento.
- La terminación tiene ámbito de árbol en todas las plataformas (grupos POSIX desprendidos con respaldo de hijo directo; Windows `taskkill /T`): `terminate()` — el único verbo de terminación — escala SIGTERM→gracia→SIGKILL (idempotente, también impulsado por la señal de cancelación del spec, no-op una vez que el árbol desaparece), y `waitForExit(signal?)` observa la vivacidad de todo el árbol para que una escalera de desmontaje propiedad del Consumer mantenga cada nivel en la quiescencia real — el gestor reacciona pero nunca clasifica el porqué (los llamadores son dueños de los plazos, las escaleras de desmontaje y la clasificación de causas).
- `spawnTerminal(spec)` es el único primitivo sin pipe. Su manejador es dueño de un PTY real, E/S de texto UTF-8, inspección/señalización del grupo de procesos en primer plano y una única operación `terminate()` esperada que alcanza la quiescencia de cada miembro de la sesión que el provider aún puede observar y asienta las llamadas de manejador en vuelo; los providers documentan los límites de observabilidad específicos del sustrato. La señal del spec solo cancela la asignación; el manejador publicado es dueño de su ciclo de vida. El flujo de salida termina tras la salida en cola cuando el proceso de nivel superior sale, y un fallo de transporte en vivo rechaza `done`. Estas operaciones siguen siendo un único primitivo de sustrato porque los pipes ordinarios no pueden asignar una terminal de control ni limpiar los miembros de una sesión de terminal; la disponibilidad, el scrollback y la política del propietario permanecen en el Consumer de PTY.
- `scrubbedParentEnv()` / `SENSITIVE_ENV_PATTERN` son la única definición compartida de depuración: los nombres ambientales con forma de credencial y los `DSH_*` se descartan, y el `env` explícito se fusiona después de la depuración. Tanto los spawns ordinarios locales como los de terminal la aplican; los transportes gestionados por el SDK que son dueños de su spawn pueden importarla directamente.
- La liberación del servicio termina todos los procesos gestionados aún en ejecución y espera su salida.

Véanse la [página de subsistema de subprocess](../../../docs/subsystems/subprocess.es.md) y el [Agent Note del seam](../../../.agents/notes/implemented/architecture/2026-07-26-subprocess-seam.es.md).

## Experiencia del modelo

Indirectamente, a través de los Consumers (hoy la familia de ejecutores bash detrás de `dsh-tool-bash`), que son dueños de toda la representación orientada al modelo de la salida y el ciclo de vida del proceso.

#### Efecto en KV Cache

Sin invalidación directa; los Consumers nombrados son dueños de cualquier cambio en el prefijo de solicitud.

## Limitaciones conocidas y trabajo diferido

- **Los spawns gestionados por el SDK quedan fuera** — un transporte del SDK que es dueño de su spawn interno no puede enrutar esa llamada a través de este servicio; aún puede importar `scrubbedParentEnv` para que la política de entorno siga teniendo una única fuente.
- **Las escaleras de desmontaje son propiedad del Consumer** — el seam entrega verbos de señalización y la espera de vivacidad del árbol, no una secuencia de quiescencia prefabricada; cada Consumer fuera de proceso codifica él mismo la forma de cooperación de su hijo (la escalera de EOF-de-stdin-primero del backend ACP es la plantilla del repositorio).
