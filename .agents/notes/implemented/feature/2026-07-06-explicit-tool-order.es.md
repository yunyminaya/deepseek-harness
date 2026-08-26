# Agent Note: Orden explícito de herramientas visible para el modelo

Status: implemented

[English](2026-07-06-explicit-tool-order.md) | Español

## Problema

El orden de herramientas visible para el modelo seguía el orden de registro de plugins, que depende de la carga concurrente de módulos para plugins por lo demás independientes. Esa carrera producía cabeceras de solicitud distintas en CI y en las grabaciones de instantáneas. Como el orden afecta a los bytes de la solicitud, al caché y a la cabecera duradera, necesita una política explícita y determinista.

## Decisión

El ensamblaje del system prompt posee el orden canónico de herramientas visible para el modelo, exactamente donde ya posee el orden de secciones. `toolOrder?: string[]` en `dsh-system-prompt` es la política explícita opcional:

- Una herramienta listada que está registrada toma su posición listada.
- Un nombre listado sin herramienta registrada es un error de configuración. Los errores de forma (entrada rest ausente o nombres duplicados) fallan desde el constructor del servicio; un nombre no registrado rechaza cada `assemble()` — el momento más temprano en que existe el conjunto de herramientas registradas contra el que comprobar (los plugins de herramientas se registran después de que el servicio se construya), y el único universal (los registros pueden cambiar en cualquier momento; cordis no tiene evento de «todos los plugins cargados»). Bajo el loop embarcado, el primer turno falla antes de cualquier solicitud de modelo — véanse las consecuencias más abajo para el radio de impacto exacto.
- Una herramienta registrada ausente de la lista se inserta en la entrada rest `'<unlisted-tools>'` (`TOOL_ORDER_REST`), en orden lexicográfico de nombre entre las demás herramientas no listadas.
- Ninguna herramienta recopilada puede usar `TOOL_ORDER_REST` como su `ToolSchema.name`; el ensamblaje rechaza ese nombre reservado antes de ordenar.
- La lista debe contener la entrada rest exactamente una vez y ningún nombre duplicado.
- Cuando `toolOrder` no está fijado, el orden canónico es el orden lexicográfico simple de nombres (comparación por unidades de código, independiente de la localidad), de modo que el determinismo no requiere configuración.

`assemble()` canoniza las herramientas del provider antes del waterfall `system-prompt/assemble`, eliminando la varianza del orden de registro en su origen. El waterfall parte de esta lista determinista; el orden sin cambios fluye entonces a la cabecera de la solicitud, a la solicitud congelada y a las comprobaciones de reconstrucción sin lógica de orden específica del loop.

El alcance es deliberadamente estrecho: esto arregla la carrera de ORDEN DE REGISTRO, no el comportamiento de los plugins. Un listener de `system-prompt/assemble` puede seguir añadiendo, quitando o recolocando herramientas — igual que puede editar secciones después de su orden — y posee el determinismo de lo que emite; el contrato del waterfall ya exige listeners deterministas (el invariante de reconstruibilidad atraparía a un listener que divergiera entre el build y el replay).

La fontanería de configuración sigue el precedente de `persona`, y `toolOrder` se sienta a su lado: las configs de las apps TUI, Headless y ACP aceptan la clave y la reenvían a través de `dsh-agent-spine-demo` (cuyo schema es la intersección de los schemas de los propietarios) al hijo `SystemPrompt`. Una nota al pie de schemastery es estructural: un array de schemastery tiene por defecto `[]`, pero un `toolOrder` omitido debe permanecer AUSENTE (= lexicográfico) en lugar de convertirse en una lista vacía configurada explícitamente (inválida — le falta la entrada rest), así que cada schema de la cadena fuerza el valor por defecto a `undefined`.

## Alternativas consideradas

- **El orden de registro (el statu quo)** — una carrera de imports concurrentes, dependiente del host (el flake de CI citado arriba), invisible en revisión.
- **Una linealización del grafo de dependencias de plugins** — la relación es parcial y los plugins de herramientas independientes son incomparables; el flake ocurrió con el orden parcial totalmente satisfecho.
- **Un `weight` por plugin en cada contribución de herramienta** — dispersa el orden por los plugins y aun así necesita una convención de numeración global que nadie posee (las bandas de `order` de las secciones muestran ese coste de coordinación pagado a mano).
- **Ordenar en `ToolRuntime.schemas()` (la capa de registro)** — igualmente determinista, pero el registro es un almacén de pertenencia que consume más que el ensamblaje; el orden es un asunto de composición de prompts, y el ensamblaje ya posee la política de composición de las secciones.
- **Una config de `LlmRuntime` + un método `orderTools()` que el loop llama antes de registrar la cabecera** — funciona, pero añade un método de servicio público y una edición del loop solo para aplicar una política a distancia; todo futuro componedor de solicitudes debe acordarse de la llamada. Canonizar donde nace la lista hace irrepresentable una lista desordenada, con cero API nueva.
- **Normalizar dentro de `llm.stream()`** — corre después de que se registre el evento de cabecera (el flake sobrevive) y reconstruye el sobre profundamente congelado, desarmando silenciosamente el invariante de reconstrucción.
- **Una lista exhaustiva (sin entrada rest)** — cada plugin de herramientas recién cargado rompería el boot; la entrada rest obligatoria mantiene las herramientas no listadas deterministas y su posición explícita.
- **Una pasada de validación en el boot (un `SystemPrompt.assertToolOrderSatisfied()` llamado por `dsh-app-boot` tras `loader.await()`)** — convertiría la mala configuración en una muerte al arrancar en lugar de un fallo en el primer turno, pero cuesta un método de servicio público más un acoplamiento estructural del pegamento genérico de boot a un servicio, y de todos modos no puede sustituir a la comprobación en el ensamblaje (los llamadores embebidos nunca corren el app boot; los registros cambian después del boot). Tampoco ningún evento existente puede alojar la comprobación: cordis v4 no tiene evento tipo ready, `loader/entry-init`/`internal/status` disparan a mitad de carga (en carrera contra el registro de herramientas, la misma entropía que mata esta Agent Note), y los eventos del ciclo de vida del agent no son anteriores al ensamblaje. Se juzgó que un único punto de aplicación en `assemble()` valía el momento de fallo más tardío.

## Consecuencias

- Todo ensamblaje construido por el registro arranca con un orden de herramientas determinista en cada host; salvo un listener experto que lo cambie deliberadamente, cada evento `request/header` y solicitud de modelo heredan ese orden. El flip de orden CI-vs-local está estructuralmente eliminado, y el valor por defecto es lexicográfico.
- El `PromptAssembly.tools` inicial es canónico, así que los listeners del waterfall parten del orden visible para el modelo; el orden de registro del provider no es observable en ningún sitio antes de ese punto de extensión cooperativo.
- El único fixture de cabecera fijado de la suite de instantáneas (`text-turn`) lleva el nuevo orden canónico de herramientas; todas las demás instantáneas ACP mantienen el cuerpo de cabecera fregado como `{{system}}`/`{{tools}}`, según el diseño de cabecera fijada.
- Una reordenación pura de herramientas entre pasos se registra como cualquier otro cambio de cabecera: una instantánea completa de `request/header` con razón `'change'`. El orden canónico estable evita que el tiempo de registro cree tales cambios en la vía ordinaria.
- La clave `toolOrder` viaja por la cadena de reenvío app → `agent-core` → `SystemPrompt`, así que los despliegues la fijan junto a `persona` en la config de la app; `dsh-llm` y el agent loop quedan intactos.
- Un nombre de herramienta mal escrito o no cargado en `toolOrder` falla el turno en el ensamblaje del prompt, no en el boot: el loop ensambla dentro del turno (tras `turn/start`, antes de `step/start`), así que el rechazo llega al catch externo del turno — el turno se cierra equilibrado con una razón `error` que lleva el mensaje, `agent/error` lo refleja, no se abre ningún paso, no se registra ningún `request/header`, ninguna solicitud llega al adaptador, y el agent vuelve a idle. Cada turno falla idénticamente hasta que se arregle la config; el proceso mismo sigue en pie (cumpliendo la regla del repo de que las referencias de configuración explícitas no deben ignorarse en silencio — el punto de aplicación es el ensamblaje porque no existe ningún momento universal anterior).
- Un provider de herramientas que devuelve el nombre reservado de la entrada rest tiene la misma forma de fallo de ensamblaje de prompt que un nombre listado desconocido. Esto impide que el centinela se convierta en una herramienta real ambigua y preserva el contrato de orden «nunca se cae una herramienta».

## Pruebas

Los tests del system prompt cubren el orden lexicográfico por defecto, la colocación listada/rest, la independencia del orden del provider, los nombres compartidos, las listas inválidas, los nombres desconocidos o reservados, la lista canónica previa al waterfall y la regla de que las herramientas añadidas por listeners no se reordenan. Los tests del loop fijan el orden registrado y despachado idéntico a través de permutaciones de registro, el reenvío por agent-core y ambas apps, las solicitudes profundamente congeladas y el fallo equilibrado de turno sin paso, cabecera ni llamada al adaptador para un nombre configurado desconocido. El replay de instantáneas conserva la lista canónica completa solo en la cabecera fijada `text-turn`; los demás fixtures siguen usando `{{tools}}`.
