# @deepseek-ai/dsh-tool-cordis

[English](README.md) | Español

El conjunto de herramientas Cordis autorreferencial: cinco herramientas orientadas al modelo sobre el runtime vivo en el proceso DSH actual. El registro, el sandbox de vm y la difusión de navegador pertenecen a [`@deepseek-ai/dsh-cordis-host-runner`](../cordis-host-runner/README.es.md) (`ctx.dynamic`), que este conjunto inyecta — una composición con estas herramientas pero sin runner nunca las activa. Hogar de diseño — semántica de sandbox, ciclo de vida y composición de paquetes dinámicos, decisiones permanentes: [la Agent Note del conjunto de herramientas](../../../.agents/notes/implemented/feature/2026-07-08-self-referential-cordis-toolset.es.md).

## Qué hace

Dos verbos emparejados, más el informe de solo lectura.

- `cordis_inspect` — informe de solo lectura sobre el proceso actual: servicios, todas las fibras de plugin vivas, herramientas registradas, los paquetes dinámicos de esta sesión, las referencias `api` / `events` respaldadas por reflexión y la superficie de slot `client` en tiempo de compilación en la que una mitad de navegador puede contribuir UI. Un `name` exacto con `what: "api"`, `what: "events"` o `what: "client"` estrecha el informe y añade el contrato completo.
- `cordis_define` — registra un paquete (`name`, `purpose`, y una mitad de host `code` y/o una mitad de navegador `client`) después de verificar la sintaxis de ambas mitades. Nada se ejecuta; el usuario ve una tarjeta en la conversación con un control de inicio. El id acuñado `dyn-<n>` viaja tanto en el valor del resultado como en los metadatos de presentación durables, que es como esa tarjeta dirige los verbos de ejecución en la reproducción.
- `cordis_run` — evalúa la mitad de host en el sandbox y entrega la mitad de navegador a cada página web abierta. Ejecutar un paquete ya en ejecución vuelve a entregar la versión viva en lugar de fallar, que es como una página recargada lo recupera.
- `cordis_stop` — dispone la mitad de host hasta la quietud y retira la mitad de navegador; la definición sobrevive y puede volver a ejecutarse.
- `cordis_undefine` — detiene el paquete si es necesario y olvida la definición; su tarjeta permanece en la conversación como un registro no cargado.

Schemas exactos orientados al modelo: [el catálogo de herramientas generado](../../../docs/tool-catalog.es.md).

Los paquetes dinámicos viven solo en la memoria compartida del proceso DSH. Permanecen activos a través de turnos posteriores y pueden afectar a otras sesiones en ese proceso, pero desaparecen después de `cordis_stop`/`cordis_undefine`, de la descarga del conjunto de herramientas o de un reinicio de DSH. No crean ningún archivo de Plugin, no instalan ningún paquete, no cambian ninguna `cordis.yml` ni configuración personal/de proyecto, no sobreviven al reinicio y no se pueden promover automáticamente. Para conservar un experimento, pide al Agent que implemente un Plugin normal local, de proyecto o de repositorio a través del flujo de desarrollo habitual. Cada verbo está limitado a la sesión: un paquete es visible y controlable solo en la sesión que lo definió.

## Postura de confianza

El sandbox aísla los globales pero no es un límite de seguridad. Los globales de Node están ausentes o redirigen a servicios de Cordis como `ctx.fs`, `ctx.web` y `ctx.bash`, y las escrituras en `globalThis` permanecen locales, pero los helpers del reino del host hacen posible la evasión. Los plugins montados reciben una fachada sin los internos del framework, pero sus servicios permitidos afectan al runtime vivo. Los schemas de herramientas dinámicos y las anotaciones cruzan el reino a través de clonación JSON iterativa y normalización de schemas, así que las declaraciones profundas válidas están limitadas por memoria en lugar de por pila de llamadas; los registros con claves invisibles al JSON y los arrays de schemas subclasados o decorados se rechazan antes de la normalización. Trata este conjunto de herramientas como un acceso a bash; véase el [diseño y la postura de confianza](../../../.agents/notes/implemented/feature/2026-07-08-self-referential-cordis-toolset.es.md).

## Configuración

Ninguna. El límite de evaluación de vm (`vmTimeoutMs`) y la ventana de acuse del navegador (`ackTimeoutMs`) pertenecen al servicio runner que es dueño del sandbox y de la difusión — véase [`@deepseek-ai/dsh-cordis-host-runner`](../cordis-host-runner/README.es.md#config).

## El catálogo de slots de cliente generado

`src/client-catalog.ts` describe los asientos de la mitad de navegador, generado por `scripts/gen-client-catalog.ts` (con puerta de frescura mediante `pnpm run verify-client-catalog` en `doc-sync`) a partir de un escaneo léxico de cada fusión de declaraciones `SlotMap` y cada sitio de llamada `slots.register`. Lleva la única superficie sobre la que una mitad de navegador puede actuar — las claves de slot, las opciones de cada llamada de registro, las props que recibe un componente, quién ocupa ya el asiento y qué mount del propietario hace existir el asiento — como datos planos: este paquete permanece del lado del host y no importa ningún módulo de cliente, así que las cadenas son lo único que cruza. El generador falla ruidosamente en lugar de publicar una entrada sobre la que un modelo no pueda actuar: un slot sin prosa orientada al registrante, un `kind`/`scope` no literal, props de propietario que ninguna exportación proporciona, una clave duplicada o un registro en un slot no declarado rompen todos la puerta. Las props del propietario se expanden un nivel — la declaración del propietario con su propia documentación de miembros, y los nombres de las formas a las que sus campos refieren — y el informe completo de un slot está limitado por presupuesto, porque estrechar a un solo slot existe para gastar menos contexto, no más.

El texto de enseñanza de un slot es el JSDoc de su declaración, así que mejorar lo que lee el modelo significa editar el contrato en su paquete declarante — no este catálogo.

## De dónde viene el informe de API

`cordis_inspect what:"api"` / `what:"events"` renderiza `src/api-catalog.ts`, la proyección generada de las declaraciones de Cordis del workspace: firmas de método renderizadas, JSDoc de origen, eventos del harness con sus modos de despacho y las formas de tipo a las que esas firmas refieren, todo producido por el mismo recorrido de AST que `docs/subsystems`, así que los datos que lee un modelo y los docs renderizados no pueden divergir. Es un hecho en tiempo de compilación sobre el REPOSITORIO, así que `pnpm run gen-cordis-api` lo regenera y `pnpm run verify-cordis-api` controla su frescura.

`src/inspect.ts` interseca ese catálogo con el almacén de servicios VIVO: lo que está EN EJECUCIÓN viene del almacén, lo que cada servicio PUEDE HACER viene del catálogo, y un servicio vivo que el catálogo no cubre se informa como alcanzable sin firmas en lugar de omitirse. Un paquete que necesita la lista en su propio código la copia de un informe — el catálogo es un hecho en tiempo de compilación sobre el repositorio, así que una lista copiada y una leída de nuevo dicen lo mismo para cualquier despliegue concreto.

Dos juicios orientados al modelo viven en este paquete en lugar de en los artefactos, porque los datos de reflexión son fieles al código mientras que un informe tiene que ser útil:

- **Solo se muestran los métodos invocables.** Los miembros que no son métodos son estado más que un verbo, y su forma renderizada lleva inicializadores del cuerpo de la implementación; los miembros con clave de símbolo son seams internos entre plugins que una fachada de paquete deliberadamente no puede alcanzar, así que nombrar uno anunciaría una llamada que no se puede hacer.
- **Solo se nombran a un modelo las claves que una mitad de host puede alcanzar.** El modelo de reflexión cubre cada `ctx.<key>` que un paquete declara, incluidos los valores de arranque proporcionados por el launcher (`agent`, `headlessIo`, …) y los servicios de la mitad de navegador (`connection`). `src/curation.ts` clasifica el `reach` de cada uno — `injectable`, `not-a-service` u `other-face` — y solo las claves `injectable` llegan a un informe: nombrar una clave que un paquete no puede alcanzar anuncia una llamada que no se puede hacer. La clasificación se lleva como datos en cada entrada del catálogo en lugar de aplicarse al renderizar, así que la exclusión es comprobable por sí sola, y `verify-cordis-catalog` fija el conjunto clasificado exactamente a las claves que la proyección de documentación no renderiza — una clave recién declarada detiene la puerta en lugar de invitar silenciosamente a un modelo a `inject` algo que nunca llegará. Una clave clasificada que sin embargo tiene un provider vivo se sigue informando como en ejecución e inyectable: el almacén de servicios es la autoridad sobre lo que existe.

El `INHERITED_CTX_API` generado cierra el informe `api` con la superficie `ctx` heredada del framework (`ctx.on`, `ctx.effect`, `ctx.loader`, los helpers de temporizador): esos miembros son el propio Context más que claves de servicio, y el nivel del framework vive en paquetes vendor fijados fuera de cada cara analizada, así que el generador cura ese nivel y lo renderiza tanto en este catálogo como en `docs/cordis-api/inherited.md`. Un servicio vivo que el catálogo no describe se informa como en ejecución y aún inyectable en lugar de como ausente. Los informes amplios de `api` / `events` renderizan resúmenes y firmas solo; un `name` exacto opta por el JSDoc de método/evento retenido, y los objetivos de servicio desconocidos o no en ejecución fallan ruidosamente.

## Renderizado

Cada herramienta renderiza una tarjeta `generic` (`read` / `execute` / `delete`); `cordis_define` lleva las mitades enviadas como `rawInput` y titula la tarjeta con la etiqueta y el propósito. Los presentadores son funciones puras de los args, y los resultados conservan el renderizado de texto predeterminado. Un cliente Web registra su propia fila `cordis_define` con clave (`@deepseek-ai/dsh-client-ui-cordis`) y lee la etiqueta, el propósito y el id acuñado de los argumentos de la llamada y de los metadatos del resultado; la tarjeta genérica es lo que usa una superficie sin ese registro.

## Forma de exportación

Plugin de namespace: exports nombrados `name` / `inject` / `apply`, sin export por defecto ([docs/postmortem/0001](../../../docs/postmortem/0001-acp-default-export-drops-inject.es.md)). Inyecta `tools` y `dynamicCordisRunner`.

## Model Experience

### Schemas de herramientas

#### Lo que ve el modelo

El modelo de conversación ve los [schemas generados de `cordis_inspect`, `cordis_define`, `cordis_run`, `cordis_stop` y `cordis_undefine`](../../../docs/tool-catalog.es.md#deepseek-aidsh-tool-cordis) siempre que este plugin sea visible.

#### Efecto de tokens

Costo de schema fijo en cada petición en esa vista de herramienta.

#### Efecto de KV Cache

Estable de prefijo mientras esta vista de herramienta no cambie. Los cambios de ámbito o de ciclo de vida del plugin que oculten estas definiciones pueden invalidar la reutilización desde el primer token de schema cambiado.

### Historial de llamadas de herramienta y resultados

#### Lo que ve el modelo

Inspect une las secciones seleccionadas exactamente como `## <section>` seguido de un salto de línea y el cuerpo dependiente de los datos, con una línea en blanco entre secciones; `what: "temporary"` usa el encabezado `## Dynamic Packages`. Cada fila informa el id, la etiqueta, el propósito, qué mitades existen, el estado de ejecución y la revisión, los servicios proporcionados y esperados, los métodos de host registrados y el último informe de carga de la mitad de navegador. El estado vacío explica que las definiciones viven solo en la memoria de este proceso. Los informes amplios de API/eventos omiten el JSDoc; `name` con `what: "api"`, `what: "events"` o `what: "client"` devuelve un objetivo exacto con su contrato completo. La sección `client` lista un asiento por línea con su cardinalidad, ámbito, resumen y si registrar allí reemplaza la UI incluida, y después las reglas transversales de registrante; las opciones de registro por asiento, las props de propietario y de framework y el ejemplo ejecutable llegan solo bajo un `name` exacto. Define responde que el paquete está definido y NO en ejecución todavía con el id para ejecutarlo; run informa la revisión, qué proporciona o espera la mitad de host y si una página acusó la mitad de navegador; stop y undefine acusan en una línea. Cada negativa es un error de herramienta que lleva el texto de enseñanza del runner. El programa enviado permanece en el historial de llamadas de herramienta del asistente.

#### Efecto de tokens

La salida de inspect y el código de paquete enviado dependen de los datos y se reenvían hasta la compactación; los acuses de ciclo de vida son pequeños. La sección `client` está limitada por el número de slots incluidos (dos líneas cada uno) y su detalle por asiento es opcional, así que el informe predeterminado crece con la superficie de slots más que con su documentación.

#### Efecto de KV Cache

Solo de adición; el contenido recién visible sigue al prefijo de petición reutilizable y no invalida las entradas existentes de KV cache.

### Peticiones posteriores después de cordis_run

#### Lo que ve el modelo

Un paquete en ejecución puede registrar herramientas, contribuciones de prompt o listeners que cambien peticiones posteriores para los ámbitos a los que apunta; `cordis_stop` y `cordis_undefine` retiran esas contribuciones después de la quietud.

#### Efecto de tokens

El impacto indirecto de tokens equivale a las contribuciones del paquete en ejecución y dura solo su ciclo de vida local al proceso.

#### Efecto de KV Cache

Ejecutar o detener una contribución de prompt o de herramienta cambia los prefijos de petición posteriores y puede invalidar la reutilización desde la primera contribución cambiada; un conjunto en ejecución sin cambios permanece estable de prefijo.

## Limitaciones conocidas y trabajo diferido

- **El sandbox es contención para código honesto, no un límite de seguridad** — los helpers del reino del host en el global del sandbox son alcanzables, así que el código de paquete puede llegar a Node; carga este plugin tan deliberadamente como concederías una herramienta bash (véase § Postura de confianza).
- **La fachada `ctx` no expone ningún `effect()`** — el código de paquete no puede registrar un disposer a medida; `on`/`provide`/`tools.register` son las vías de limpieza admitidas.
- **Los límites de vm y de acuse pertenecen al runner** — véanse sus [Limitaciones conocidas](../cordis-host-runner/README.es.md#known-limitations-and-deferred-work); un cuerpo de mitad de host asíncrono escapa a `vmTimeoutMs`.
