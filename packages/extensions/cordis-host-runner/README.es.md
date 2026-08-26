# @deepseek-ai/dsh-cordis-host-runner

[English](README.md) | Español

La mitad de host de los paquetes dinámicos montados por el modelo: el registro de definiciones, el sandbox `node:vm` y el ciclo de vida del fiber para las mitades de host, la tabla de manejadores invoke, y el viaje de ida y vuelta de ejecución que lleva a cabo una página de navegador. Se aporta como `ctx.dynamicCordisRunner`. Las herramientas orientadas al modelo viven en [`@deepseek-ai/dsh-tool-cordis`](../tool-cordis/README.es.md); la mitad de navegador la carga [`@deepseek-ai/dsh-cordis-client-runner`](../cordis-client-runner/README.es.md).

## Qué hace

Dos fases: `define` solo registra, y todo lo que tiene efecto cuelga de una ejecución.

- `define` / `undefine` son dueños de la vida de una definición. `define` recorta y exige los metadatos, precomprueba la sintaxis de cada mitad compilándola (sin ejecutar nada), acuña `dyn-<n>` y registra la definición contra la sesión que la pidió — no tiene ningún efecto que deshacer, de modo que el código no analizable se rechaza antes de que exista un id. `undefine` detiene primero una definición en ejecución y luego la olvida. Ninguna cruza el cable: solo define la llamada de herramienta del propio modelo.
- `run` responde a la solicitud del modelo de ejecutar una definición, y sus dos formas difieren según de quién sea el asunto del paquete. Un paquete solo de host es asunto de este proceso: la mitad de host se evalúa en la vm bajo el fiber de grupo `cordis-dynamic` y la llamada vuelve. Un paquete con mitad de navegador tiene que llevarlo a cabo una página, de modo que `run` se convierte en un viaje de ida y vuelta respondible — emite `cordis/request-run`, se suspende, y lo resuelve una persona al permitirlo o rechazarlo. No hay temporizador; el `AbortSignal` de quien llama (el turno que preguntaba fue cancelado) es la única otra salida, y anuncia la cancelación para que las demás páginas dejen de ofrecer una respuesta. Si alguna página responderá no es conocible cuando se envía la solicitud — una página que la recibió puede no responder nunca, de modo que un despliegue sin página conectada se suspende como cualquier otra solicitud sin responder y termina en `cancelled`. `run` no tiene cara de cable — `cordis_run` lo llama en proceso.
- `runHostHalf` / `getClientCode` son los pasos que recorre una página permitida, primero la mitad de host, de modo que un fallo de la mitad de host corta el flujo antes de que el navegador se haya movido. `runHostHalf` es idempotente por contrato: un paquete en ejecución se enlaza en lugar de volver a evaluarse, las llamadas concurrentes para una definición la evalúan una vez, y `startedHere` nombra a quien la hizo. `getClientCode` entrega entonces a esa única página el código fuente de la mitad de navegador, rechazando una definición que ya no existe, que no tiene mitad de navegador o que no está en ejecución. El código nunca viaja en un anuncio, de modo que esta es la única forma de que llegue a un navegador.
- `resolveRequestRun` cierra el viaje de ida y vuelta con el veredicto de la página que responde, y transmite `cordis/request-run-resolved` para que todas las demás páginas suelten la afordancia pendiente. Gana la primera respuesta; un id de solicitud posterior o desconocido se acepta y se ignora. Un éxito que nombra una revisión que el registro ya ha superado se rechaza en lugar de aplicarse (`accepted: false`, la solicitud sigue suspendida), porque la página que respondió cargó un dispatch que ya no está en vivo. Un veredicto de fallo deshace la mitad de host solo cuando esta misma solicitud la evaluó, de modo que una página que no puede cargar su propia mitad nunca detiene un paquete que usan las demás páginas.
- `stop` deshace un dispatch en vivo — se eliminan los manejadores, el fiber de la mitad de host se libera hasta la quiescencia, se transmite `dynamicCordisRunner/retract` — y deja la definición ejecutable.
- `inventory` responde el registro completo, sin segmentarlo por sesión y con cada fila nombrando la sesión propietaria, porque la superficie de control de ejecución es global. Enumerar no es actuar: cada verbo de acción sigue comprobando esa propiedad. Cada fila indica además si la definición tiene mitad de navegador, de modo que una superficie de control de ejecución solo ofrece cargarla en la página actual cuando hay una mitad que cargar. `snapshot` es su contraparte local al host y con ámbito de sesión, que lleva el fiber de cada mitad de host en vivo para que `cordis_inspect` pueda renderizar provides/waiting/state por sí mismo (un fiber no puede cruzar el cable).
- `reportRenderFailure` registra lo que una página vio hacer mal a una mitad de navegador CARGADA en el momento del renderizado. El renderizado ocurre estrictamente después de que una carga tenga éxito, de modo que una ejecución ya ha respondido `ok` para entonces: este informe es de disparar y olvidar, no tiene autoridad de resolución y nunca toca `resolveRequestRun` ni ninguna parte del resultado de la ejecución — **no es el `report`/ack v2 retirado**. El host conserva el último fallo por definición entre todas las páginas (un segundo informe de otra página lo sobrescribe), y una ejecución nueva, una detención o un undefine lo limpian, de modo que al modelo nunca se le muestra un fallo de un dispatch que ya no existe. La cara de la mitad de navegador conserva su propio «lo que ESTA página muestra ahora»; las dos responden preguntas distintas en lugar de duplicar una. Se descarta un informe de una definición que la sesión que informa no posee, porque la ruta de informes no debe fallar jamás un renderizado.
- `invoke` enruta una llamada de la mitad de navegador de un paquete a un método que su propia mitad de host registró con `harness.handle`. La infraestructura solo enruta — no existe dirección de host a navegador.

Un rechazo de `run` o `stop` nombra uno de `definition-missing`, `host-half-failed`, `client-half-failed`, `rejected`, `cancelled` o `not-running`; los tres últimos son respuestas, no defectos — una persona rechazó, el turno que preguntaba terminó, o no había nada en ejecución que detener.

Una definición que definió otra sesión se lee como ausente, no como prohibida, de modo que nada se filtra entre sesiones. `invoke` y `resolveRequestRun` no llevan sesión alguna: la llamada de un componente y la respuesta de una página son hechos globales a la página, no de una sesión.

Cuatro eventos reenviados pertenecen a esta funcionalidad, declarados por este paquete en su subruta [`./types`](src/types.ts) segura para el cliente e incluidos en la lista de permitidos para su entrega por [`@deepseek-ai/dsh-api-remotes`](../../api/remotes/README.es.md), que es lo que permite a un navegador alcanzarlos a través de `ctx.remote.$on`: `cordis/request-run` (`{requestId, agentId, id, name, purpose}` — metadatos, nunca código), `cordis/request-run-resolved` (`{requestId, outcome}`), `dynamicCordisRunner/package` (`{id, name, rev}`) y `dynamicCordisRunner/retract` (`{id, rev}`). Los dos últimos son un par simétrico que anuncia el estado de ejecución — cada arranque nuevo y cada detención, tenga o no el paquete mitad de navegador.

## Postura de almacenamiento

El registro es memoria de proceso y la única fuente de verdad. El log de sesión lleva los metadatos de una llamada de define — nunca su código — de modo que un proceso reiniciado legítimamente no tiene definiciones, y una tarjeta cuyo id ya no se resuelve lo dice exactamente en lugar de fingir que puede ejecutarse. Nada de esto se escribe en disco y ninguna definición se restaura automáticamente; una página recargada no contiene nada hasta que alguien ejecuta un paquete de nuevo, que es lo que hace que enlace la mitad de host en vivo y vuelva a obtener la mitad de navegador.

## Postura de confianza

El sandbox de la vm aísla los globales pero no es un límite de seguridad: los globales de Node están ausentes o redirigen a servicios de Cordis (`ctx.fs`, `ctx.web`, `ctx.bash`, los helpers de temporizador), y una mitad de host recibe una fachada sin los internals del framework, y sin embargo los servicios que declara alcanzan el runtime en vivo. Trata un paquete dinámico como acceso a bash — consulta la [Agent Note del toolset autorreferencial](../../../.agents/notes/implemented/feature/2026-07-08-self-referential-cordis-toolset.md).

## Config

| Campo | Predeterminado | Significado |
|---|---|---|
| `vmTimeoutMs` | `5000` | Milisegundos que la porción síncrona de una mitad de host puede ejecutarse en la vm antes de que se aborte la evaluación |

Hay un solo campo: una solicitud de ejecución espera a una persona, de modo que el viaje de ida y vuelta no tiene plazo propio.

## Forma de exportación

Paquete de servicio: exporta por defecto `DynamicCordisRunnerService` (clave de servicio `dynamicCordisRunner`), con `./types` llevando las formas de payload que comparten el namespace remoto `dynamicCordisRunner` y sus consumidores. Las formas de `define` / `undefine` permanecen dentro del paquete, porque nunca cruzan el cable.

## Experiencia del modelo

### Rechazos y errores didácticos retransmitidos por las herramientas de cordis

#### Qué ve el modelo

Nada directamente: este paquete no registra ninguna herramienta ni inyecta ningún prompt. Sus rechazos llegan al modelo a través de los resultados de las herramientas `cordis_*` que lo llaman — una mitad no analizable nombra la línea infractora, una definición ausente explica que las definiciones viven solo en memoria, una ejecución `rejected` o `cancelled` reporta que una persona rechazó o que el turno terminó, no que algo fallara, y una carga fallida de la mitad de navegador lleva el texto de error propio de la página que responde.

#### Efecto de tokens

Ninguno propio: cada mensaje anterior lo lleva el resultado de la herramienta que llama.

#### Efecto de KV Cache

Una mitad de host que registra herramientas cambia la vista de herramientas de la siguiente solicitud, lo que invalida la reutilización del prefijo desde el primer token de schema cambiado; ejecutar o detener un paquete sin registros de herramientas es neutro para el prefijo.

## Limitaciones conocidas y trabajo aplazado

- **Una ejecución con éxito no significa que la interfaz se haya renderizado.** `run` vuelve en cuanto la página que responde ha CARGADO la mitad de navegador; React renderiza después, de modo que un componente que lanza una excepción no puede aparecer en el recibo de ejecución. El fallo aflora a través de `reportRenderFailure` y se lee de vuelta con `cordis_inspect what:"temporary"`; el resultado de la ejecución lo dice en lugar de dar a entender éxito.

- Un paquete con mitad de navegador **se suspende cuando no hay ninguna página conectada** — los despliegues headless y ACP mantienen la ejecución en espera hasta que se cancela el turno que preguntaba, porque un evento reenviado no informa de quién lo recibió. Los paquetes solo de host no se ven afectados.
- Una solicitud de ejecución suspendida **no tiene plazo**: espera a una persona hasta que se cancela el turno que preguntaba, de modo que la automatización sin supervisión no puede usar paquetes con mitad de navegador.
- `vmTimeoutMs` acota solo la evaluación síncrona; un cuerpo de mitad de host asíncrono se le escapa, en línea con la postura de confianza cooperativa del toolset.
- `runHostHalf` no lleva id de solicitud, de modo que «qué solicitud evaluó esta mitad de host» se atribuye del lado del host a la solicitud armada más recientemente para esa definición; varias solicitudes de ejecución concurrentes para una definición necesitarían revisar esa regla.
- Una respuesta de éxito que nombra una revisión superada se rechaza (`accepted: false`) y deja la solicitud suspendida, de modo que la llamada del modelo termina solo con una respuesta válida o con su propia cancelación. Resolverla requeriría una orquestación nueva contra la revisión en vivo, y ninguna página hace eso hoy — la [mitad de navegador](../cordis-client-runner/README.es.md) no lee el ack — de modo que en la práctica esa solicitud la cierra la respuesta de otra página o la cancelación de quien llama.
- El `inject` declarado de una mitad de navegador se lee del plugin que devuelve en la página, de modo que el anuncio no lleva ningún campo de declaración de servicios.
- **`zod` es una dependencia de runtime de las caras TypeRT generadas, no de `src`.** `./typert` y `./remote` resuelven a `lib/typert.*.js`, que `tsc` emite sin empaquetar con un `import { z } from 'zod'` directo, de modo que el paquete debe declararlo (el precedente de `@deepseek-ai/dsh-goal`) y `knip.json` debe ignorarlo para este workspace — knip lee el código fuente, y estas caras son productos de compilación. Nada en `src` importa zod.
