# @deepseek-ai/dsh-cordis-client-runner

[English](README.md) | Español

Mitad de navegador de los paquetes dinámicos de dos mitades. El runner del lado del host guarda el código de cada definición en la memoria del proceso y pregunta a las páginas abiertas, a través de un evento `cordis/request-run`, si quieren ejecutar una; este paquete responde a esa solicitud, convierte la definición en un plugin de navegador en vivo, y convierte un evento `dynamicCordisRunner/retract` de vuelta en una página limpia.

## Qué hace

1. **Suscripción de eventos** — los cuatro anuncios son eventos de cordis del host reenviados, de modo que este paquete consume `cordis/request-run`, `cordis/request-run-resolved` y `dynamicCordisRunner/retract` a través de `ctx.remote.$on`, cuyo conjunto de claves ES la lista de permitidos de api-remotes.
2. **Evaluación de closures** — el código fuente de la mitad de navegador se ejecuta como cuerpo de una función asíncrona cuyos parámetros son su superficie de símbolos (`React`, `console`, `styles`, `host`, más trampas didácticas que sombrean `setTimeout`/`fetch`/`require`). Nada de JSX, nada de TypeScript, nada de imports de módulos.
3. **Fachada de guarda** — `apply` recibe un proxy de lista blanca sobre el ctx real del fiber: los verbos de ciclo de vida más los servicios que el plugin devuelto declaró en su propio `inject` (de modo que la forma de objeto `{ inject: ['slots'], apply(ctx) {} }` es lo que llega a un servicio; una función simple no tiene lugar de declaración y no alcanza ninguno). El asiento `slots` asigna la prioridad de sombreado (registrar ES sombrear, gana la ejecución más reciente); el asiento `theme` fija el origen de la capa de anulación al id del paquete y cuelga su liberador en el fiber.
4. **Entradas del loader** — el plugin protegido se asienta en la tabla de módulos y se monta a través de `loader.create`, de modo que un paquete dinámico pasa por las mismas compuertas de activación, la misma limpieza de efectos del fiber y la misma proyección de estado que uno estático. La descarga es eliminación de la entrada más invalidación de la fábrica más eliminación de estilos.
5. **Orquestación de la ejecución** — un evento `cordis/request-run` pregunta a esta página si quiere ejecutar una definición. Quien responda dirige la ejecución en orden: primero la mitad de host, luego la obtención del código fuente, luego la mitad de navegador y, por último, una resolución que cuenta lo sucedido. Que una persona pulse «run» es en sí mismo la autorización y orquesta de la misma manera, sin nada que responder — y para una definición solo de host la ejecución termina en la mitad de host, porque aquí no hay una segunda mitad que obtener ni cargar.
6. **RPC interno del paquete** — el `host.call` de un paquete se enruta a su propia mitad de host a través del namespace remoto `dynamicCordisRunner` (`invoke`), y cada código de fallo de enrutado se convierte en su propio error didáctico. Ambas direcciones transportan solo JSON: un argumento omitido viaja como `null` (de modo que `host.call('listServices')` es legal y el manejador recibe `null`), y un payload que el codec generado rechaza — una función, `undefined`, una instancia de clase — se convierte en un error didáctico que nombra la llamada y el contrato en lugar del nombre desnudo del campo del codec.
7. **Redistribución del fallo de renderizado** — el seam de supervisión del registro de slots (`slots.onEntryError`) se dispara ante cada fallo en el límite de entrada de la página; los que pertenecen a un paquete que este runner asentó van desde esa única observación a dos salidas: aguas arriba hacia la sesión autora (`reportRenderFailure`, para el modelo) y hacia el campo `renderFailures` propio de este paquete (para la fila del panel). La propiedad se determina por la identidad del componente, registrada cuando el proxy `register` de la guarda lo asienta, porque el registro almacena el componente tal cual — de modo que no hace falta mantener sincronizado un registro paralelo de entradas. Esto es solo diagnóstico posterior a la resolución: no tiene autoridad de resolución, nunca toca la resolución de una ejecución, y un informe fallido se traga en lugar de convertir un fallo en dos.

## Ciclo de vida

Las cargas convergen por `(id, rev)` frente al estado en vivo: cargar una revisión que esta página ya ejecuta responde desde el estado en vivo sin recargar (de modo que una ejecución reproducida no parece sin responder), una revisión más reciente la reemplaza, y la misma revisión después de una retracción se carga de nuevo. Las operaciones se serializan por definición.

Nada se carga en la activación y nada se restaura tras una actualización — una página solo ejecuta un paquete dinámico cuando alguien responde a una solicitud de ejecución o lo pide aquí.

## Qué lee y llama una superficie de ejecución

`ctx.dynamicCordisRunner` es toda la cara:

- `activeRuns` — la única actividad en curso de cada definición: `awaiting-approval` (el id de solicitud que responder más la sesión, el nombre del paquete y el propósito de la petición) u `orchestrating` (la sesión para la que se lleva a cabo la ejecución). Ambos brazos nombran la sesión porque la agrupación pertenece a la ejecución, no a su fase; el brazo en espera lleva el texto de la propia petición porque `cordis_define` no transmite nada, de modo que una solicitud puede nombrar una definición que la última lectura del registro no cubre, y entonces esta entrada es la única fuente que tiene esa fila. Una superficie renderiza a partir de ella y no guarda ninguna copia, que es lo que hace que la afordancia sobreviva a un remontaje.
- `renderFailures` — el último fallo de renderizado de esta página por definición (slot, mensaje didáctico y si el fallo retiró la entrada de su celda), en el mismo canal de notificación que el conjunto en vivo. Local a la página y actual por construcción: se limpia cuando el paquete se detiene, se retracta o se carga de nuevo, de modo que una fila puede renderizarlo directamente. El host conserva su propia copia del último fallo entre todas las páginas para el modelo — las dos tienen propietarios y ciclos de vida distintos, y una superficie no debe leer la del host en lugar de esta.
- `lastRunError` — por qué falló el propio intento de esta página, por definición. Sobrevive a la actividad, porque el host solo libera la mitad que inició una solicitud fallida: una página puede estar mirando una definición que el host reporta como en ejecución sin tener nada cargado ella misma.
- `approve(requestId)` / `decline(requestId)` / `startUserRun({ agentId, id, hasClientHalf })` — las dos entradas. Las tres son idempotentes (por id de solicitud, y por definición en la ejecución propia de la persona usuaria), de modo que una doble pulsación no puede iniciar dos ejecuciones. `hasClientHalf` es obligatorio: una definición solo de host no tiene código fuente que obtener, de modo que quien llama declara la forma a partir de la fila del registro sobre la que actúa, en lugar de que el orquestador la descubra con una obtención fallida. Una solicitud respondible siempre tiene mitad de navegador, porque el host ejecuta él mismo una definición solo de host en lugar de preguntar a una página.
- `subscribe()` / `getSnapshot()` / `isLoaded(id)` — lo que esta página tiene cargado. `isLoaded` es la verdad local de la página, nunca el «está en ejecución» del host.

## Experiencia del modelo

### Resolución de ejecución, cuando un modelo pidió la ejecución

#### Qué ve el modelo

Este paquete no aporta ninguna herramienta, prompt ni contexto propios; lo primero que crea y llega a un modelo es la resolución que devuelve para un viaje de ida y vuelta `cordis/request-run`, que el host convierte en el resultado `cordis_run` bloqueado. Un éxito lleva la revisión cargada y, para una mitad de navegador aparcada en servicios que esta página no tiene, sus nombres. Un fallo lleva un único motivo — `rejected` cuando la persona rechazó, `host-half-failed` o `client-half-failed` — y, para la mitad de navegador, el texto propio de este paquete: la etapa que falló (`evaluate`, `module-import` o `activate`) seguida del mensaje de la closure, la guarda o el fiber. Los errores didácticos de la guarda (un servicio no declarado, un global de navegador sombreado, un plugin que no devolvió ningún `apply`) llegan al modelo exactamente a través de ese campo. Un fallo que ocurre más tarde, mientras React renderiza la mitad cargada, viaja por la ruta separada de post-resolución que se describe abajo.

#### Efecto de tokens

Condicional y acotado: como máximo una resolución por solicitud de ejecución, gastada dentro del resultado de la herramienta `cordis_run` que el host ya emite. El texto depende de los datos (el mensaje de error de la propia definición) y este paquete no conserva nada entre solicitudes — los fallos de carga posteriores de una página son diagnósticos locales de la página sin portador visible para el modelo.

#### Efecto de KV Cache

Solo anexión. Una resolución llega al modelo únicamente como resultado de la herramienta para la solicitud que ya estaba en curso, extendiendo la cola del historial; nada de lo que crea este paquete reescribe ni reordena los tokens de solicitudes anteriores, de modo que un prefijo que por lo demás sería reutilizable sigue siéndolo. Las ejecuciones repetidas de la misma definición producen cada una su propio resultado en lugar de reemplazar uno anterior.

### Fallo de renderizado, después de resolverse la ejecución

#### Qué ve el modelo

Una mitad de navegador que carga limpiamente puede fallar igualmente cuando React la renderiza, y ese fallo aterriza después de que la ejecución se respondiera — de modo que, de otro modo, al modelo se le diría «ok» y nunca se enteraría. Cada fallo en el límite de entrada de un paquete que esta página asentó se envía al host (`reportRenderFailure`) nombrando el slot, si el fallo retiró la entrada de su celda (`abdicated`: la interfaz del paquete ha desaparecido, no solo está rota), y un mensaje escrito para la persona autora: el texto del fallo, más la corrección para un global de navegador retenido que el texto nombra pero no explica — `window.setInterval` alrededor de la trampa de la closure falla como `is not a function`, que por sí solo no explica nada. El host conserva el último por paquete y lo muestra a través de `cordis_inspect`; nada de esto llega a la resolución de una ejecución. La misma observación también aterriza en `renderFailures` para la superficie propia de la página — un observador, dos salidas, porque «el último fallo entre todas las páginas, para el modelo» y «lo que esta página muestra ahora» son hechos distintos con ciclos de vida distintos.

#### Efecto de tokens

Condicional y acotado por la retención del host, no por esta página: un informe por fallo, y el host conserva solo el último por paquete, de modo que una entrada que falla repetidamente le cuesta al modelo un párrafo en lugar de una lista creciente. El informe nunca entra en un resultado de herramienta propio — el modelo solo paga por él cuando lo pregunta.

#### Efecto de KV Cache

Ninguno propio. Los informes viajan por RPC y se almacenan, no se anexan a la conversación; el modelo los lee a través de una inspección que decidió hacer, que extiende la cola como cualquier otro resultado de herramienta.

## Limitaciones conocidas y trabajo aplazado

- **Una resolución rechazada no se reintenta.** El acuse de recibo de `resolveRequestRun` no se lee, de modo que cuando el host rechaza un éxito obsoleto (`accepted: false`, porque la revisión de la definición avanzó mientras esta página cargaba), la página conserva lo que cargó y no vuelve a orquestar. La solicitud sigue siendo respondible — la respuesta de otra página o la cancelación de quien llama la resuelven — y la detención que avanzó la revisión retracta la carga obsoleta. El reintento se evaluó y se aplazó: la ventana es un único avance de revisión dentro de un solo viaje de ida y vuelta.
- El plugin declara `remote.dynamic`, de modo que permanece aparcado hasta que exista el namespace del lado del host en lugar de cargar paquetes cuya mitad de host nunca podría alcanzar.
- La admisión de slots (listas de permitidos/denegados por despliegue) no tiene portador: la fila despachada declara servicios, no slots de destino.
- Las listas blancas de la guarda son gemelos espejados a mano de la fachada del sandbox del lado del host; compartir una única especificación queda aplazado.
