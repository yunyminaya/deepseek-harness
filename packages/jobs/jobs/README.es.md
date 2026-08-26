# @deepseek-ai/dsh-jobs

[English](README.md) | Español

El contrato del registro de trabajos en segundo plano (`ctx.jobs`). El `JobRegistry` abstracto y sus tipos de vocabulario dan a los producers de larga duración ids compartidos, aislamiento de propietario, lecturas, cancelación, espera, avisos y limpieza bajo un único contrato; el registro local al proceso vive en [`dsh-jobs-local`](../jobs-local/README.es.md). Los plugins productores extienden `JobKindMap` con su namespace de ids opaco.

## Contrato del servicio

- `start(spec): JobId` valida el controlador adjunto, el spec, el propietario vivo exacto, el `outputLimitBytes` positivo opcional y cualquier política de admisión propiedad del provider antes de llamar una vez a `run()` del producer. Un rechazo de preflight o un throw del arrancador no deja ningún id de trabajo ni trabajo registrado; un retorno correcto confirma sin otro paso fallible.
- `get(id, caller?)` y `list(caller?)` devuelven instantáneas que no consumen. El listado incluye solo los trabajos propiedad del llamador y los que no tienen propietario.
- `read(id, caller?)` consume el único cursor de los trabajos de stream y lee la salida terminal de forma idempotente para los trabajos de salida final.
- `kill(id, caller?, reason?)` invoca la cancelación del producer antes de cambiar el estado. Un throw de cancelación deja el trabajo en ejecución; el éxito lo cambia a `stopping` y marca la entrega terminal como informada.
- `wait(id, timeoutMs, caller?, signal?)` devuelve una instantánea terminal o la instantánea viva al agotar el tiempo. Abortar detiene solo la espera; el asentamiento gana una vez que ha confirmado la entrega terminal a quien esperaba.
- `onJobDone(listener)` observa cada registro terminal con el propietario exacto. Los throws y rechazos de los listeners están contenidos; el trabajo del listener no se espera.
- `onJobsChanged(listener)` observa los cambios del conjunto visible — registro, cada transición a stopping (la del teardown incluida, antes de que espere a un producer lento), asentamiento, eliminación por disposición del propietario y los commits de vaciado de la disposición del servicio — y lleva solo el propietario cuyo conjunto se movió, o `undefined` cuando cambió un trabajo sin propietario y el conjunto de todos los llamadores se movió con él. Es granular por propietario porque la eliminación es un cambio que ningún registro por trabajo puede expresar, y no es un superconjunto de `onJobDone`: no lleva significado de entrega y no marca nada como informado. El registro se vincula al fiber que llama, así que un observador montado fuera del registro ve igualmente el vaciado de la disposición.
- `attachController(name)` declara un controlador de trabajos para la vida de su effect. `start()` falla antes de la ejecución del producer cuando ningún controlador adjunto atiende al propietario del spec.

Los tres registros son relativos al propietario, porque un único registro atiende a todas las composiciones del proceso. Un controlador o listener registrado desde un contexto sin ámbito atiende a todos los propietarios; uno registrado bajo el ámbito de la composición de un agent atiende exactamente a los agents compuestos bajo ella. Así, una composición que no carga ningún controlador no puede iniciar trabajo en segundo plano apoyándose en los controles de otra composición, y un asentamiento notifica solo a los listeners que registró la composición de su propietario.

El acceso con propietario compara el `SessionId` del trabajo con el del llamador. Los ids como `bash-1` son predecibles, así que esta valla es el límite. Los trabajos sin propietario están abiertos a los llamadores y duran hasta la disposición del servicio.

`outputLimitBytes` es política de presentación al modelo propiedad del producer que se traslada sin cambios a las instantáneas. Un controlador lo aplica después de añadir los metadatos de estado o de aviso; el registro no reescribe la salida del producer ni inventa un valor por defecto para los producers que lo omiten.

Las implementaciones deben también la semántica de ciclo de vida del contrato: los registros sobreviven a los fibers de producers y controladores, la disposición de propietario y de servicio cancela el trabajo vivo y espera a los producers conformes, y el asentamiento es de primero que gana — un registro terminal, una ronda de notificación contenida a los listeners, quienes esperaban liberados.

Ver el [catálogo de tipos de trabajos](../../../docs/subsystems/jobs.es.md), la [Agent Note del runtime](../../../.agents/notes/implemented/architecture/2026-06-20-generic-long-running-tool-runtime.es.md) y la [Agent Note del seam](../../../.agents/notes/implemented/architecture/2026-07-26-job-registry-seam.es.md).

## Experiencia del modelo

Indirectamente, a través de los plugins productores y de [`dsh-tool-jobs`](../tool-jobs/README.es.md), que renderizan los ids de trabajo, la salida, el estado, la cancelación y los avisos de finalización.

#### Efecto en la caché KV

Sin invalidación directa; el consumer nombrado es dueño de cualquier cambio en el prefijo de solicitud.

## Limitaciones conocidas y trabajo diferido

- **La salida de stream tiene un único cursor consumidor** — los observadores independientes necesitan una API de cursor o de instantánea.
- **El trabajo en primer plano no se puede promocionar** — los producers eligen primer plano o segundo plano antes de empezar.
- **El contrato es intra-proceso** — `JobStart.run()` pasa callbacks y objetos `Agent` exactos; un backend duradero o entre procesos debe remodelar la identidad, el reinicio, la propiedad y la semántica de observación antes de poder implementar este seam.
