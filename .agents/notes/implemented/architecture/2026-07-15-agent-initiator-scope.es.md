# Agent Note: El ámbito del agent iniciador sobre AsyncLocalStorage

Status: implemented

[English](2026-07-15-agent-initiator-scope.md) | [中文](2026-07-15-agent-initiator-scope.zh.md) | Español

## Problema

El harness tiene dos nociones de contexto útiles pero distintas. Un `Context` de Cordis selecciona servicios, propiedad de registros y ciclo de vida; `agent.ctx` es el scope de registro plano propiedad de un Agent vivo. La identidad de Agent y de Session describe en cambio el sujeto de una operación asíncrona. Cambiar el significado de un `ctx.agent` raíz para que signifique «el Agent que esté ejecutándose» mezclaría esos significados y fallaría cuando un proceso impulsa Agents de forma concurrente.

La infraestructura profunda local al proceso a veces necesita un Agent iniciador de confianza por debajo de los parámetros explícitos de loop, herramienta y petición — por ejemplo, un transporte consciente del host, una helper de trazado, un logger o un cliente de gateway. Exigir que toda helper privada reenvíe `agent` añade repetición, mientras que una ranura mutable global al proceso es incorrecta a través de `await`. Los argumentos visibles para el modelo no sirven, porque un modelo no debe poder elegir una Session de confianza ni una cabecera de enrutamiento. El carrier pertenece al servicio de Agent, no a un contexto opcional visible para el modelo.

## Decisión

El servicio obligatorio `ctx.agents` usa `AsyncLocalStorage` de Node para transportar el Agent iniciador. Almacena el `Agent` exacto directamente en lugar de introducir una frame de un solo campo; un token de ejecución privado separado registra el linaje de los límites anidados solo para la contabilidad del teardown y no transporta identidad. El [catálogo de core-data](../../../../docs/subsystems/core.es.md#initiating-agent) identifica el tipo transportado.

`currentInitiator()` lee de forma opcional, `requireInitiator()` lanza `no initiating agent is active`, y `withInitiator(agent, operation)` preserva el valor síncrono o la Promise exactos de la operación. `withoutInitiator(operation)` establece un límite de despeje para el trabajo que no debe heredar un Agent. La Session sigue derivándose como `agent.session`; el turno, el paso, la llamada de herramienta, la `signal`, el modelo, `cwd`, el sandbox y la autorización permanecen en sus propietarios existentes.

`AgentLoop` ya inyecta `ctx.agents` y envuelve el ciclo de vida completo de `runLoop` de cada driver concreto en `agents.withInitiator(agent, ...)`. Sus entradas de orquestación de loop, turno, paso y llamada de herramienta privadas al paquete recuperan el Agent exacto de `ctx.agents`, derivan `agent.session` una vez y dejan que las helpers locales a la operación lo capturen en lugar de reenviar el driver concreto o la `Session` a través de interfaces superficiales. Una helper hoja mantiene un parámetro `Session` estrecho cuando esa es su interfaz real, en lugar de aceptar un `Context` más amplio solo para una búsqueda ambiental.

Los drivers concurrentes reciben stores independientes. Las continuaciones de un driver hijo transportan al hijo, mientras que el llamador reanuda en su store anterior en cuanto `withInitiator()` devuelve; el seguimiento de ejecuciones activas mantiene la Promise devuelta en el drenaje del teardown hasta que se liquida. La creación, la carga de persistencia y el `setup(agentCtx)` no publicado quedan fuera del límite del driver hijo: la creación iniciada por un padre se ejecuta bajo la identidad del padre, mientras que `agentCtx.agent` identifica explícitamente al hijo.

La identidad ambiental no reemplaza los contratos explícitos. `ToolExecution.agent`, `AssembleContext.agent`, `GenerateOptions.sessionId`, la propiedad de trabajos, las peticiones padre/hijo, `ctx.agent`, `agentCtx.agent`, los sujetos de aprobación y de hooks, la selección de `cwd`, la cancelación, los mensajes de worker/proceso, los registros de persistencia y la identidad de wire siguen siendo explícitos. Un límite remoto materializa en su petición tipada la identidad que necesita, porque ALS es local al proceso.

`AgentRegistry` posee un ciclo de vida de iniciador ordenado. El teardown primero rechaza nuevos límites; quitar `ctx.agents` drena entonces los dependientes inyectados como AgentLoop, y el registro espera a los límites activos de Promise devuelta antes de llamar a `AsyncLocalStorage.disable()`. Si la cadena asíncrona heredada de un límite inicia el unload de un fiber de Cordis propietario, el linaje privado de tokens de ejecución libera esa cadena de límites anidados del drenaje, lo que impide que el teardown espere sobre sí mismo mientras límites no relacionados siguen drenando. `currentInitiator()` y `requireInitiator()` siguen siendo utilizables a través de una referencia de servicio en vuelo retenida mientras transcurre el drenaje ordinario; después de la disposición, los métodos de iniciador lanzan `agent initiator scope is disposed`. La disposición del Context raíz puede iniciar el teardown de fibers hermanos de forma concurrente, por lo que el recuento de límites activos sigue siendo necesario además del ordenamiento de dependencias de Cordis.

El scope del iniciador no es dueño del trabajo desacoplado: el drenaje del registro solo sigue la Promise devuelta por `withInitiator()` o `withoutInitiator()`. Los recursos asíncronos creados dentro de un límite heredan su store hasta que se liquidan o ALS se desactiva, por lo que su seam propietario debe detener explícitamente el trabajo no devuelto. El trabajo en primer plano propiedad del agent devuelve su ciclo de vida y mantiene su contrato de cancelación. Los timers, colas e infraestructura de despliegue no relacionados arrancan bajo `withoutInitiator(operation)`; los límites de cola, worker, proceso y wire serializan la identidad en lugar de esperar propagación de ALS.

Un transporte consciente del host puede derivar una cabecera propiedad del despliegue como `X-Harness-Session-Id` de `ctx.agents.requireInitiator().session.id`; la cabecera está ausente del schema y de los argumentos visibles para el modelo. Ningún transporte MCP o Web de producción adopta tal cabecera en esta decisión. Un transporte doble de prueba demuestra el límite de confianza sin asignar política de enrutamiento del host a un seam existente neutral al provider.

Esta decisión extiende el [contrato de scope de registro de agent](2026-07-08-agent-scope-contexts.es.md) y su [diseño de runtime](2026-07-12-agent-scope-runtime-design.es.md); no cambia su significado estático de `agent.ctx`.

## Verificación

Las pruebas del servicio de agent fijan las lecturas opcionales y obligatorias, la identidad síncrona exacta y la identidad de Promise entre realms, la observación del settlement de Promise intrínseco, los límites solapados, anidados y despejados, la restauración tras throws o rechazos, el orden de drenaje ordinario y reentrante, y los errores de referencia retenida. La integración de AgentLoop fija los drivers concurrentes y anidados, las llamadas sin agent, el reinicio de AgentRegistry, el teardown raíz y la programación de loop y herramienta privada al paquete a través de la búsqueda ambiental. Las comprobaciones de composición, grafo de módulos, build y cierre de runtime mantienen `ctx.agents` conectado a través del bundle por defecto, la columna vertebral del SDK, el cierre de runtime de Python y los harnesses directos de AgentLoop sin otro provider.

Un transporte doble consciente del host deriva `X-Harness-Session-Id` internamente y verifica que el schema de herramienta y los argumentos registrados no contienen ningún campo de identidad. El servicio deliberadamente no drena trabajo asíncrono omitido de la Promise devuelta por la operación de límite; ese trabajo sigue sujeto al contrato de detención explícito de su propietario.

## Alternativas consideradas

**Pasar Agent por cada función.** Los límites públicos, de worker, de proceso, de persistencia y de wire siguen haciéndolo, pero exigir que toda helper privada local al proceso transporte Agent añade reenvío repetitivo sin mejorar la confianza. ALS queda confinado a la cadena asíncrona dentro de esos límites explícitos.

**Hacer `ctx.agent` dinámico.** `ctx.agent` ya significa el Agent estático asociado a un contexto Cordis con ámbito de agent. Cambiar el significado raíz mezclaría los scopes de registro y de ejecución y haría sorprendente el comportamiento concurrente.

**Añadir un servicio separado `ctx.agentExecution`.** El carrier no tiene backend, configuración ni tipo de identidad independientes: almacena el mismo `Agent` que `ctx.agents` ya posee, y AgentLoop ya depende de ese servicio. Un segundo provider obligatorio añadiría cableado de paquete, composición, ciclo de vida, catálogo generado y harness de pruebas sin separar una capacidad real.

**Almacenar una frame de runtime con nombre o completa.** Una frame de un solo campo `{ agent }` solo envuelve el valor, mientras que Agent, Session, inbox, cancelación, turno, paso, ejecución de herramienta y persistencia ya tienen propietarios autoritativos. Añadir más campos crearía instantáneas obsoletas y otro ciclo de vida; transportar `Agent` directamente mantiene el límite nombrado por sus métodos sin duplicar estado.

**Incluir un `AbortSignal`, `cwd`, sandbox o autorización de paso.** Sus ciclos de vida y autoridad no coinciden con el límite del driver, y sus seams existentes ya los pasan explícitamente. Añadir una capacidad de control requiere una decisión separada y un contrato de ciclo de vida anidado.

**Usar un `currentAgent` global al proceso.** Los Agents y subagentes concurrentes se sobrescriben unos a otros a través de continuaciones en espera, así que un global mutable solo es correcto bajo una garantía de serialización que el harness no ofrece.

**Derivar la identidad de argumentos visibles para el modelo.** La entrada del modelo o del usuario no puede ser de confianza para seleccionar el enrutamiento de Session, tenant o sandbox.

**Añadir identidad de enrutamiento a cada seam de capacidad.** Eso reparte preocupaciones de hospedaje a través de APIs neutrales al provider. Una implementación consciente del host es dueña de su cabecera de transporte mientras los límites públicos siguen siendo explícitos.

## Consecuencias

La infraestructura profunda gana un Agent iniciador de confianza local al proceso sin ampliar las peticiones existentes de herramienta y capacidad. Los drivers concurrentes y anidados se aíslan automáticamente, AgentLoop no gana ningún servicio obligatorio adicional, y la disposición de HMR/raíz alcanza la quiescencia antes de que ALS se desactive.

La dependencia es implícita en las firmas de función y transporta un objeto Agent con capacidad. Los consumidores deben restringirla a infraestructura transversal, tratar la presencia ambiental como ni vivacidad ni autorización, y conservar comprobaciones explícitas de cancelación y propiedad. ALS también tiene un coste de propagación siempre activo y no cruza límites de worker, proceso, HTTP ni cola durable.

El diseño de teardown acepta deliberadamente la dependencia de `AsyncLocalStorage.disable()` de Node, de [Stability 1 (Experimental)](https://nodejs.org/api/async_context.html#asynclocalstoragedisable). Node exige `disable()` antes de que una instancia de ALS pueda ser recolectada, lo que importa cuando HMR reemplaza instancias propiedad de AgentRegistry; el guard de estado del servicio impide que un límite posterior reentre en la instancia después de la disposición.

El scope transporta deliberadamente solo el Agent, omitiendo turno, paso, `signal`, `cwd`, sandbox y autorización. Un consumidor real que no pueda usar los campos explícitos existentes debe justificar por separado cualquier refinamiento; un campo copiado obsoleto puede a lo sumo etiquetar mal telemetría, nunca conceder control.
