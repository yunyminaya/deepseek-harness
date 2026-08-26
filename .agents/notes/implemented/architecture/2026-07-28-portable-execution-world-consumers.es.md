# Agent Note: Consumers portables sobre los mundos de ejecución de filesystem y subprocess

Status: implemented

[English](2026-07-28-portable-execution-world-consumers.md) | Español

## Problema

Los seams de filesystem y subprocess hicieron reemplazables el acceso a archivos y a procesos ordinarios, pero PTY y LSP seguían llegando directamente a las APIs de Node del host. Por tanto, un provider de ejecución remota parecía necesitar paquetes PTY y LSP separados aunque su comportamiento de dominio no cambiara. Esos paquetes serían adaptadores superficiales: cada uno duplicaría un Consumer existente solo para reemplazar sus operaciones de archivo y de proceso.

Un mundo de coding remoto solo es útil cuando las operaciones de archivo, los comandos, los terminales y los servidores de lenguaje comparten una identidad de sandbox. Mover el harness completo a ese sandbox enredaría además la experimentación con providers con la carga de plugins, las credenciales, el transporte de modelo, la durabilidad de sesión, la supervisión y el despliegue.

Los pipes ordinarios no cubren un requisito. Un terminal persistente necesita asignación de PTY, inspección y señalización del grupo de procesos en primer plano, y limpieza de la sesión de terminal completa. Pretender que esas operaciones pueden reconstruirse en `dsh-terminal-bash` a partir de un handle de `spawn()` ordinario filtraría los internals del provider o debilitaría su contrato de ciclo de vida.

## Decisión

`ctx.fs` y `ctx.subprocess` definen juntos un mundo de ejecución. Los providers montados juntos deben describir el mismo namespace de rutas, los mismos ejecutables, procesos y sesiones de terminal; las capacidades superiores consumen esas dos interfaces en lugar de nombrar al provider.

La interfaz de filesystem es dueña de los hechos de ruta que otra capacidad necesita sin exponer su identidad de destino opaca: una ruta de proceso canónica, un URI `file:` canónico y la contención. Las operaciones de texto completas y en streaming existentes siguen siendo propiedad del filesystem; los Consumers de protocolo aplican sus propios límites de retención mientras consumen el stream.

La interfaz de subprocess es dueña de la búsqueda de ejecutables y de las primitivas de proceso: el spawn de procesos ordinarios, crudos o recogidos, y `spawnTerminal()`. La operación de terminal es una única primitiva profunda cuyo handle es dueño de la E/S de texto, de los grupos en primer plano, de la señalización y de una operación esperada TERM-a-KILL que resuelve las llamadas de handle en vuelo y alcanza la quiescencia para todo miembro de sesión que el provider aún pueda observar. Su señal solo cancela la asignación; el handle publicado es dueño de su propio ciclo de vida. La detección de prompt, la inferencia de inactividad, el scrollback, la política de sandbox y el ciclo de vida del dueño permanecen en el Consumer de PTY.

Los Consumers genéricos usan ese mundo de ejecución:

- `dsh-bash-local` sigue mapeando la semántica de Bash sobre el `ctx.subprocess.spawn()` ordinario.
- `dsh-lsp-stdio` lee y contiene source a través de `ctx.fs`, resuelve y lanza servidores de lenguaje a través de `ctx.subprocess`, y transporta los URIs de archivo propiedad del provider a través de la inicialización y el renderizado de resultados. Una señal de ciclo de vida del provider aborta el trabajo de filesystem y de protocolo durante el disposal, incluida la búsqueda de workspace antes de la propiedad de la cola; su JSON-RPC, pooling, sincronización y normalización permanecen sin cambios.
- `dsh-terminal-bash` mapea la semántica de shell persistente sobre `ctx.subprocess.spawnTerminal()`. La implementación local de `node-pty` y de inspección de procesos se mueve a `dsh-subprocess-local`; otro provider de subprocess suministra la misma primitiva. `danger-full-access` no necesita `ctx.sandbox`; un modo confinado requiere un provider de sandbox del mismo mundo y falla antes del spawn cuando no hay ninguno montado. La evidencia de prompt y de silencio recogida durante la inspección asíncrona previa a la escritura se descarta cuando comienza la escritura del provider. La cancelación conserva la reserva de envío mientras una escritura en vuelo se resuelve y luego señala al grupo en primer plano, de modo que los bytes tardíos o la señal no pueden alcanzar a un sucesor; un sondeo de preparación en vuelo no puede liberar esa reserva, y una escritura rechazada no envía señal. El deadline absoluto permanece armado durante toda la cancelación. Un fallo de señal se convierte en fallo de transporte de terminal. La finalización de una inspección obsoleta reanuda el sondeo del envío actual. La cancelación al arranque comienza el rollback del terminal sin esperar a una llamada de preparación o de señalización bloqueada. Close rechaza las señales públicas nuevas y delega la quiescencia de sesión observable por el provider en la operación de terminación esperada del handle.

## Frontera del POC E2B

La realización E2B opcional tiene exactamente tres paquetes específicos de provider bajo `packages/e2b/`: `dsh-e2b` crea un sandbox y lo borra al expirar el timeout o en el disposal, `dsh-fs-e2b` implementa `ctx.fs`, y `dsh-subprocess-e2b` implementa `ctx.subprocess` sobre E2B Commands, PTYs y grupos de procesos Linux remotos. Los dos adaptadores obtienen el único handle del SDK del dueño y nunca crean sandboxes privados.

E2B es dueño del filesystem mutable, de los procesos gestionados de comando y de Bash, de la asignación de terminal y de los grupos de sesión de terminal, de los procesos de servidor de lenguaje y de las lecturas de source, y de los archivos privados del adaptador bajo `.dsh-e2b`. El host es dueño de los objetos de Cordis y de plugins, del agent loop, del estado de agent/sesión/goal, de los logs y la persistencia de sesión, de las llamadas LLM, de los prompts y las herramientas, de la autoridad, de los skills, de la orquestación de subagentes, de los buffers y la preparación de PTY, del estado de protocolo LSP y de los buffers de SDK/red de E2B. El overlay ni sube ni sincroniza el workspace del host.

Los adaptadores conservan solo la mecánica de sustrato. La canonicalización de filesystem cruza el transporte de comandos decodificado del SDK como un encuadre estricto de NUL codificado en base64; las lecturas en streaming dejan los límites de bytes en los Consumers. La salida de comandos de subprocess y las instantáneas de entorno usan ASCII/base64 donde la decodificación por chunks del SDK perdería bytes, mientras que los shells de control privados aíslan los perfiles y los lanzamientos posteriores blanquean los nombres con forma de credencial descubiertos. La limpieza de procesos y de terminales usa grupos remotos y demuestra la quiescencia antes de resolverse.

El estado del sandbox es deliberadamente efímero: el timeout y el disposal borran los archivos remotos y el estado no gestionado. El POC no añade reconexión ni retención de pausa/salida, ni backend de persistencia de sesión, constructor de plantillas, volumen, instantánea, capa de política de red, catálogo de sandbox, sincronización de workspace, handles remotos duraderos ni ejecución de todo el harness.

## Verificación

Las suites de paquetes enfocadas fijan el ciclo de vida del sandbox, el encuadre de rutas canónicas, los metadatos de filesystem y las versiones atómicas, la publicación/rollback de subprocess, la E/S de texto de terminal y la limpieza de sesión, los límites de salida, la cancelación, el disposal y el registro de invariantes. Una composición de Loader con credenciales ejercita el mismo provider de tres paquetes a través de imports de source y exports compilados, incluidos la visibilidad FS/Bash, los perfiles de login hostiles, la salida UTF-8 dividida en bytes, la limpieza de procesos y terminales, las consultas LSP, el aislamiento del workspace del host y el borrado final del sandbox.

## Alternativas consideradas

**Mantener un paquete PTY y LSP por provider remoto.** Rechazada porque la mecánica de provider se repetiría por encima de los seams existentes. El test de borrado expone el problema: borrar esos adaptadores no debería dispersar el comportamiento de dominio en el provider remoto; los Consumers genéricos ya son dueños de él.

**Crear un sandbox separado por capacidad o herramienta.** Rechazada porque las operaciones de archivo y de proceso no compartirían identidad ni estado, frustrando el caso de uso de coding y multiplicando los dueños de ciclo de vida.

**Modelar un terminal como un subprocess ordinario con pipes.** Rechazada porque los pipes no pueden asignar un terminal de control, resolver el grupo de procesos en primer plano actual ni demostrar una limpieza completa de la sesión de terminal. Una única primitiva de terminal es más pequeña y más honesta que exponer escapes hatch específicos del sustrato.

**Mover la preparación de PTY y la política de sesión al servicio de subprocess.** Rechazada porque son semántica de Consumer de terminal persistente, no mecánica de procesos del SO. Un provider de subprocess es dueño de lo que solo su sustrato puede hacer; `dsh-terminal-bash` es dueño de lo que significa un terminal de Harness.

**Exponer operaciones separadas de terminación y quiescencia de terminal más un controlador de ciclo de vida compartido.** Rechazada porque todo Consumer de terminal necesita el mismo resultado único de limpieza. Las operaciones separadas exportan contabilidad de provider, observador acotado y semántica de reintento sin un Consumer de producción; una única operación esperada del provider es una interfaz más profunda.

**Añadir una primitiva estable de lectura acotada al seam de filesystem.** Rechazada porque solo LSP necesita un límite de bytes de documento completo, que puede imponer mientras consume el stream de texto existente. Una segunda primitiva obliga a todo provider a implementar mecánicas de handle estable y no-follow, incluido un protocolo de helper remoto, sin un defecto observado de reemplazo concurrente.

**Ejecutar el harness completo dentro del entorno remoto.** Rechazada como un modelo de despliegue distinto. Hacer portables las capacidades de ejecución no mueve las llamadas de modelo, el estado de sesión, el estado de plugins ni el agent loop.

**Poner todas las operaciones de provider en un único paquete dueño compartido.** Rechazada porque la identidad y el ciclo de vida del sandbox son las únicas preocupaciones del dueño. Filesystem y subprocess conservan contratos, tests y Consumers distintos sin convertir al dueño en un cajón de sastre de capacidades.

**Implementar las operaciones de filesystem remoto solo mediante comandos de shell.** Rechazada porque eso descarta la identidad de filesystem estructurada, los errores, el streaming, las guardas de versión y la semántica de mutación atómica que ya consumen las herramientas de archivo.

**Añadir una abstracción genérica de runtime distribuido o reconectar handles en vivo.** Rechazada porque los seams de capacidad existentes transportan los contratos demostrados, mientras que la identidad remota por sí sola no puede reconstruir callbacks, promesas pendientes, autoridad, estado de protocolo ni cursores de salida. Una capa nueva especularía sobre persistencia y sincronización más allá del POC.

## Consecuencias

Un provider de ejecución remota implementa solo su dueño de sandbox compartido más los adaptadores de filesystem y subprocess. Bash, PTY y LSP se componen por encima, de modo que las correcciones de esas capacidades siguen siendo neutrales respecto al provider.

Las interfaces fundamentales son más amplias, y un par filesystem/subprocess debe ponerse de acuerdo en un único mundo de ejecución. Las operaciones añadidas se limitan a hechos y mecánicas de ciclo de vida que los Consumers genéricos actuales requieren; los schemas de modelo, el encuadre de protocolo, la política de preparación y la presentación no se filtran a los providers.

La implementación local absorbe `node-pty` y la inspección de procesos de la plataforma porque es dueña de la mecánica local de terminales. Esto mueve código sin debilitar el teardown del terminal: el disposal barre a los descendientes antes y después de terminar el shell de nivel superior, espera a los descendientes con identidad exacta acotada por PID retenidos durante la inspección en primer plano, y retiene a los miembros de sesión Linux que sobreviven a la salida del nivel superior. macOS no puede enumerar una sesión POSIX después de que su líder salga, así que un hijo que se re-padrea entre instantáneas de inspección sigue siendo una limitación explícita del provider local, no un motivo para devolver la mecánica de procesos al Consumer de PTY.

La composición E2B demuestra que un dueño de sandbox compartido más los adaptadores de filesystem y subprocess bastan para sacar del host el mundo de coding mutable y dejar al mismo tiempo las capacidades superiores neutrales respecto al provider. Sus límites de POC siguen siendo explícitos: el SDK retiene el transporte completo de comandos en memoria del host, el arranque remoto no puede publicar un PID de forma síncrona, los hechos exactos de espera de stdin de terminal y de señales independientes no están disponibles, las operaciones numéricas de PID/PGID no están acotadas por identidad, la sonda inicial de entorno no puede ocultar secretos desconocidos por defecto del sandbox a procesos del mismo UID ya en marcha, y los artefactos del adaptador permanecen hasta el borrado del sandbox. Son restricciones del provider, no justificación para shims de compatibilidad ni para más paquetes E2B.
