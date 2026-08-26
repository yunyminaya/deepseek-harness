# Agent Note: Presentación de herramientas por agent y el preset `code`

Status: implemented

[English](2026-08-05-per-agent-tool-presentation.md) | Español

## Problema

Los agent presets componen las herramientas de un agent (agente) por sesión, pero no la FORMA en que esas herramientas llegan al modelo. Code Mode — una herramienta `run_code` más un SDK de TypeScript generado, que sustituye una secuencia de llamadas por un solo programa — era un campo `mode` a nivel de despliegue en la fila `dsh-tools` del host. Un despliegue ejecutaba cada sesión en Code Mode o ninguna, así que la forma de producto evidente («代码模式» junto a 标准/极简/创造 en el selector de presets) no tenía dónde apoyarse.

La lectura ingenua de «mover las herramientas al plano del agent» no funciona. `ctx.tools` tiene consumidores del plano del host que no pueden seguirlo: `dsh-agent-loop` lee el seam privado de programación del registro, `dsh-apiproxy` lee sus presentadores para renderizar las tarjetas de herramienta, y cada plugin de herramientas se registra en él. Por la propia regla del stack — un servicio solo se mueve a un preset cuando TODOS sus consumidores se mueven con él — el registro permanece donde está.

## Decisión

Separar el registro de su proyección. El registro permanece en el plano del host; la **presentación** pasa a ser estado con scope dentro de él, junto a las restricciones y guardas con scope que ya viven allí.

`ToolRuntime.presentAs(mode)` es solo con scope y refleja a `restrict()`: escribe una celda en el `ToolLayer` del scope llamador a través de `ScopedLayers.effect`, así que se deshace con el scope que la declaró. En la superficie Web incluida, ese scope es el montaje permanente del preset de un agent — el preset `code` lleva la fila `tool-presentation` — así que una declaración cubre a todos los agents unidos a ese preset, y `modeFor(scope)` toma la declaración más cercana de la cadena. Se resuelve contra el `mode` de la configuración, que se convierte en el valor por defecto para los scopes que no declaran nada, en lugar de un hecho a nivel de proceso. Las tres lecturas que decidían la presentación — los wire schemas, la entrada `run_code` de la vista de visibilidad y la sección del SDK generado — toman el mode del scope en lugar del del servicio.

De ahí se siguieron dos consecuencias que soportan carga:

- **`run_code` se añade por scope.** Antes, el transporte entraba en cada vista siempre que existía. Por agent, un agent nativo no debe encontrar `run_code` en su tabla de despacho porque algún otro agent del proceso lo presenta — así que la adición depende del propio mode de ese scope, y el transporte se construye de forma perezosa en la primera necesidad.
- **El nombre reservado es ahora incondicional.** `run_code` solo se rechazaba como registro mientras había configurado un code mode. Cualquier agent puede ahora seleccionar un code mode, así que un nombre libre de tomar en un despliegue nativo se convertiría en una colisión en cuanto se montara un preset.

La sección de prompt del SDK se registra globalmente mediante un despliegue con code mode (sin cambios) y, además, por scope mediante `presentAs`, donde hace sombra por nombre. Su cuerpo se renderiza vacío para un scope nativo, y el renderizador de prompts descarta lo vacío — eso es lo que mantiene a un agent que opta por NO entrar en un despliegue con code mode libre de una sección de SDK.

El preset expresa la elección mediante una fila, `@deepseek-ai/dsh-agent-tool-presentation`, cuyo cuerpo entero es una llamada a `presentAs`. Un code mode espera a `ctx.codeRuntime` mediante `ctx.inject` en lugar de darlo por supuesto: el runtime está en el plano del host, y una fila pendiente es lo que `dsh-agent-presets` ya informa como un montaje no utilizable, nombrando la fila — así que un preset que selecciona Code Mode contra un despliegue sin runtime falla donde un operador puede actuar.

## Alternativas consideradas

**Un segundo `ToolRuntime` dentro del realm aislado del preset.** Rechazado: `dsh-agent-loop` resuelve el registro una sola vez desde el contexto del host a través de un símbolo privado, así que un registro por agent sería invisible para el programador. Hacer que el loop resuelva el registro por agent es un cambio mucho mayor que hacer que un solo campo sea consciente del scope.

**Una clave de nivel superior en el YAML del propio preset.** Rechazada por la misma razón por la que los metadatos de presentación del preset fueron a un `preset.yml` aparte: la composición es una lista de nivel superior de filas de plugin y no puede llevar claves hermanas.

**Nombrar el paquete `dsh-tool-mode`.** Rechazado por una gate, correctamente. `gen-tool-catalog` recorre con glob `packages/*/tool-*` y exige que cada coincidencia publique un schema de herramienta orientado al modelo, porque ese prefijo significa «incluye una herramienta» en este repositorio. Esta fila no incluye ninguna.

**Registrar la sección de SDK incondicionalmente desde el constructor.** Rechazado tras probarlo: `renderPrompt` descarta las secciones vacías pero `PromptAssembly.sections` las conserva, así que cada despliegue nativo llevaría una entrada `tools:sdk` que no renderiza nada, y dos aserciones existentes sobre esa lista habrían tenido que debilitarse para acomodarla.

**Compartir la composición de `standard` mediante include.** Rechazado según la propia convención del stack: `cordis` ya duplica `standard`, y el valor de un preset es que toda su composición se puede leer en un solo archivo. El costo — una tercera copia de ~240 líneas que debe moverse a la vez — es real, y es el argumento más fuerte para un futuro mecanismo de include.

## Consecuencias

Dos sesiones en un mismo proceso pueden ahora presentarse de forma distinta, así que «qué herramientas ve el modelo» ya no se puede responder solo con la configuración de despliegue; requiere el agent. Todo diagnóstico que cite un mode cita ahora el del scope, no el del servicio.

`ctx.tools.schemas(agent)` sigue siendo el catálogo de CAPACIDADES del agent y no cambia con la presentación — solo se colapsan las herramientas del assembly. Los tests que afirman lo que recibe el modelo deben leer el assembly; `web-agent-presets.spec.ts` afirma ambos lados de esa distinción para el preset `code` incluido.

La lista incluida es de cuatro presets (标准/代码/极简/创造), así que cualquier golden que los enumere se mueve. Un despliegue que no compone ningún runtime de código no puede componer ningún preset de code mode; el overlay Web incluido lleva uno, la composición base no.
