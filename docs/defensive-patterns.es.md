# Patrones defensivos

[English](defensive-patterns.md) | Español

Reglas de clases de bugs ganadas a pulso: cada patrón de abajo es una clase de defecto que llegó a publicarse aquí, o estuvo a punto de hacerlo, enunciada como la regla que evita que se repita. Lee esto antes de escribir código de ciclo de vida, concurrencia, subprocesos o teardown. Las contrapartidas a nivel de test (ruta de entrada real, verificación con el mundo real, propiedad de recursos) están en [testing.md](testing.es.md).

## Reporta los resultados ortogonales de forma independiente

Un resultado puede ser varias cosas a la vez: un proceso puede agotar el tiempo de espera Y salir con 0 porque capturó la señal. Expón cada hecho independiente (`timedOut`, `signal`, `exitCode`) por su cuenta; nunca anides el reporte de una bandera dentro de la rama de otra, o quien llama leerá una ejecución cortada como un éxito limpio.

## Respeta los contratos públicos en AMBOS lados

Cuando una implementación recibe varias representaciones de un mismo resultado, normalízalas antes de devolverlas a través de la API pública. Las implementaciones de `LlmAdapter.stream()` pueden lanzar o emitir `finish {kind:'error'|'aborted'}`, pero `LlmRuntime.stream()` expone los fallos de petición al modelo solo como chunks de finish terminales; los defectos de middleware y de los consumidores siguen lanzándose. Así los consumidores no tienen que adivinar si una excepción capturada vino del provider, de un wrapper, del logging de chunks o de su propio ensamblaje. Documenta el contrato normalizado donde se define el tipo; ejercita cada forma de origen a través del consumidor real.

## El estado asíncrono no es estado síncrono

`agent.followup()` no tiene una finalización ni un resultado por mensaje; la finalización de un trabajo en segundo plano compite con los límites de los turnos; `reader.close()` se dispara tanto para EOF como para la liberación del recurso. Nunca trates `agent/status` ni `whenIdle()` como el resultado de un solo follow-up: varios follow-ups encolados, steering (direccionamiento) y trabajo inyectado pueden compartir un mismo intervalo `running`, mientras que la cancelación o la liberación pueden descartar elementos no iniciados. Un llamador de automatización que realmente sea dueño de una ejecución debe definir su intervalo explícitamente —por ejemplo, desde la recepción durable en el inbox de su mensaje hasta el siguiente `idle` de todo el agent— y describir cualquier salida seleccionada como propia del intervalo, no atribuida causalmente a ese mensaje. El guard corta en ambas direcciones: si la transición esperada nunca puede ocurrir, la espera se cuelga, así que maneja explícitamente la rama de «nada que esperar».

## Dispose debe alcanzar la quietud, no solo pedirla

Un teardown que emite kills/aborts pero regresa antes de que el trabajo se detenga deja huérfanos. Haz la limpieza async y espera la salida de los hijos (kill → await `done`), y cierra los registros de listeners/notificaciones ANTES de matar para que las finalizaciones tardías permanezcan en silencio.

## Contén las excepciones de los callbacks en el dispatcher

Un listener proporcionado por el usuario que lanza una excepción no debe rechazar la promesa en la que se ejecuta ni dejar sin atender a los listeners posteriores. Envuelve el bucle de dispatch en try/catch y registra; un subscriber malo nunca rompe el ciclo de vida del core.

## Nunca le des a la salida no confiable el entorno ambiente ni rutas predecibles

Los comandos spawnados reciben un env depurado (elimina `*KEY*`/`*SECRET*`/`*TOKEN*`/`*PASSWORD*`) para que las credenciales del harness no puedan filtrarse a la salida, a `env` ni a los archivos de spill. Los archivos temporales/de spill usan un directorio privado (0700), nombres aleatorios y aperturas exclusivas solo del propietario (`'wx'`, `0o600`) —las rutas predecibles y legibles por cualquiera invitan a las carreras de symlinks y a la divulgación.

## Desenlaza las rutas con forma de enlace

Una ruta que puede ser un symlink o un junction de Windows se elimina con `lstatSync().isSymbolicLink()` y luego `unlinkSync`: unlink borra solo el enlace y rechaza un directorio real, así que nunca sigue el enlace hasta su destino. `rmSync(link)` de Windows lanza `ERR_FS_EISDIR` sobre un junction; la eliminación recursiva puede descender a través de uno hasta su destino. Reserva el `rmSync` recursivo para directorios reales conocidos.
