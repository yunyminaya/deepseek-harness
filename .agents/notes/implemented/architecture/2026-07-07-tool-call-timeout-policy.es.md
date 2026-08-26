# Agent Note: La política de timeout de llamadas de herramienta como plugin

Status: implemented

[English](2026-07-07-tool-call-timeout-policy.md) | [中文](2026-07-07-tool-call-timeout-policy.zh.md) | Español

## Problema

El [Agent Note de timeout/deadline](2026-07-06-timeout-deadline-library.es.md) extrajo la primitiva de temporización y clasificación a `@deepseek-ai/dsh-timeout`, pero la política de timeout seguía anclada a capacidades individuales y a schemas visibles para el modelo. `bash` exponía `timeoutMs`; `web_fetch` exponía `timeout_ms`; `web_search` no tenía timeout visible para el modelo aunque los providers ya respetan `exec.signal`; una futura herramienta grep/glob importaría directamente la biblioteca de timeout o inventaría su propia política. Esa es la forma de autoría equivocada para un SDK de plugins: un autor de herramientas debería normalmente reenviar `exec.signal` a la implementación que llama, y la política de despliegue debería decidir el presupuesto.

Al mismo tiempo, no todo timeout del repo es un presupuesto de llamada de herramienta visible para el modelo. Los hooks ejecutan hooks de comandos llamando directamente a `ctx.shell`, no a través de `ctx.tools.execute()`, y la herramienta de modelo `bash` multiplexa la ejecución en primer plano, el arranque en segundo plano, el sondeo en segundo plano y la reutilización de hooks a través del mismo backend. Trasladar cada timeout a un plugin de herramientas en un solo paso confundiría esas vías y arriesgaría romper la semántica de timeout de los hooks.

## Decisión

El timeout de llamada de herramienta es una política que se aplica solo a la ejecución de herramientas visible para el modelo, en tres partes:

- `@deepseek-ai/dsh-timeout` sigue siendo la biblioteca compartida dueña de `deadline()` y `timeoutOf()`.
- `@deepseek-ai/dsh-tools` tiene un waterfall alrededor del despacho, `tools/execute`, entre `tools/pre-execute` y `tools/post-execute`.
- El [contrato de nombres del repositorio](2026-08-11-repository-naming-contract-and-rename-ledger.es.md) nombra `@deepseek-ai/dsh-tool-call-timeout-policy` por la operación exacta que limita. El plugin lee el `timeoutMs` declarado de cada herramienta desde el runtime y envuelve una llamada que tenga uno derivando una nueva `exec.signal`.

El pipeline de ejecución es:

```text
ctx.tools.execute(exec)
  -> tools/pre-execute
  -> tools/execute
       -> registry dispatch (the base next())
            -> tool.execute(args, exec)
            -> thrown tool errors normalize to ToolExecutionResult
  -> tools/post-execute
```

El comportamiento por defecto es conservador: una herramienta que no declara `timeoutMs` no recibe ningún deadline `TOOL_TIMEOUT` del plugin.

### El punto de extensión alrededor del despacho `tools/execute`

`@deepseek-ai/dsh-tools` declara un waterfall `tools/execute` cuyo `next()` base es el thunk de despacho-con-normalización — el mismo `try`/`catch` interno que convierte una herramienta que lanza (o una herramienta desconocida) en un `ToolExecutionResult` `isError`. Un listener recibe `(exec, next)`: llama a `next()` para delegar en el despacho (devolviendo su resultado, opcionalmente envuelto) o devuelve un resultado de sustitución para cortocircuitar el despacho. Todo el pipeline sigue dentro del try/catch externo de `execute`, así que un listener que lanza se convierte en un resultado `isError`, nunca en un fallo de turno.

Que el catch sea el `next` base — y no algo externo al waterfall — es una pieza estructural: cuando un provider ve la señal de timeout y lanza su propio error de aborto upstream, el despacho del registro primero lo convierte en un resultado de error normal, y solo entonces `timeout-policy` puede sustituir el resultado final por `TOOL_TIMEOUT`.

### El plugin `timeout-policy`

El plugin es `@deepseek-ai/dsh-tool-call-timeout-policy`, un plugin de función/namespace sin configuración (`name` / `inject` / `apply`) del grupo `packages/guard/` (originalmente su propio grupo `timeout/`). El presupuesto por herramienta se DECLARA en la herramienta, no en este plugin: un `ToolDefinition` lleva un `timeoutMs` opcional, que el plugin propietario de la herramienta fija desde su propia configuración. `dsh-tool-web`, por ejemplo, resuelve `fetchTimeoutMs` / `searchTimeoutMs` (por defecto 30000) en las definiciones de `web_fetch` / `web_search`:

```yaml
- id: timeout-policy
  name: '@deepseek-ai/dsh-tool-call-timeout-policy'
- id: tool-web
  name: '@deepseek-ai/dsh-tool-web'
  config:
    fetchTimeoutMs: 30000
    searchTimeoutMs: 30000
```

Los timeouts viven en las definiciones de herramientas en lugar de en un mapa de nombres de texto libre, eliminando la política no usada por errores de escritura. `defineTool` valida un presupuesto finito positivo. Durante el despacho el aplicador deriva una señal de deadline y la asigna a `exec.signal`; el registro fusiona ese deadline con la señal original del llamador antes del cuerpo bajo el [contrato de cancelación cooperativa de herramientas](2026-07-19-cooperative-tool-cancellation.es.md). El aplicador restaura después la señal del llamador y convierte su propia expiración en `TOOL_TIMEOUT`; las herramientas sin presupuesto pasan sin cambios.

La sustitución de señal se hace por **mutación in situ de `exec.signal`**, no pasando un objeto nuevo a `next()`. El `next()` del waterfall de Cordis ignora cualquier argumento que se le pase y vuelve a invocar a los listeners aguas abajo con el array de payload compartido (`vendor/cordis/src/events.ts`), así que la mutación es como el wrapper suministra su deadline al registro. El registro re-fusiona la señal capturada del llamador inmediatamente antes del cuerpo, y el plugin restaura `exec.signal` a la original del llamador en un `finally` para que `tools/post-execute` nunca vea la señal de deadline del plugin.

`timeout-policy` es dueño de ambos usos del código `TOOL_TIMEOUT`: el código de deadline interno pasado a `deadline()`/`timeoutOf()` (con scope para que un deadline externo anidado se lea como una cancelación ordinaria) y el código de error estructurado del resultado de herramienta. Su resultado de sustitución es:

```ts ignore-check
function toolTimeoutResult(timeoutMs: number): ToolExecutionResult {
  return {
    content: [{ type: 'text', text: `Error: tool call timed out after ${timeoutMs}ms` }],
    isError: true,
    error: {
      message: `tool call timed out after ${timeoutMs}ms`,
      info: { name: 'ToolTimeoutError', code: 'TOOL_TIMEOUT' },
    },
  }
}
```

Este es un deadline cooperativo. No mata trabajo arbitrario compitiendo con la promesa de la herramienta; la herramienta o la capacidad a la que llama debe honrar `exec.signal` y alcanzar el reposo. Declarar `timeoutMs` por tanto SIGNIFICA «esta herramienta es cooperativa con `exec.signal`», que el README del plugin enuncia como su contrato.

No se necesita ningún evento de sesión nuevo para la reconstruibilidad: `TOOL_TIMEOUT` es el `tool/result` final visible para el modelo de esa llamada, así que el log de sesión existente ya registra el contenido y el error estructurado `{ name, code }` que verá la siguiente petición del modelo.

### Adaptación de herramientas existentes

`web_fetch` y `web_search` están migradas. `dsh-tool-web` conserva la propiedad de sus schemas visibles para el modelo, y esos schemas no exponen ninguna perilla de timeout: `web_fetch` no tiene parámetro `timeout_ms`, mientras que `web_search` acepta un array `queries` obligatorio sin argumento de timeout. Los cuerpos de las herramientas no importan `@deepseek-ai/dsh-timeout`; reenvían `exec.signal` a `ctx.web`.

`dsh-web-fetch-http` conserva un único `timeoutMs` configurado a nivel de provider como red de seguridad de recursos grande para los llamadores directos de `ctx.web.fetch()` y para despliegues mal configurados; no es dueño de ningún timeout visible para el modelo. Cuando una señal `TOOL_TIMEOUT` llega primero al provider de captura, la clasificación con scope de provider la trata como `WEB_ABORTED` upstream, y el wrapper externo `tools/execute` sustituye el resultado final de la herramienta por `TOOL_TIMEOUT`. Un despliegue de herramienta web publicado configura la red de seguridad del provider por encima del presupuesto de `timeout-policy` para que la política de llamada de herramienta normalmente gane para las llamadas del modelo.

`bash` permanece en la vía de timeout actual del backend. `dsh-tool-bash` sigue exponiendo `timeoutMs` y `run_in_background`; `dsh-bash-local` sigue usando `@deepseek-ai/dsh-timeout` para `BASH_TIMEOUT`; los puentes de hooks siguen llamando a `runHook()` y pasando `timeoutMs` a través de `ctx.shell`. Esto mantiene estable el comportamiento de primer plano/segundo plano/hooks.

`read`, `write`, `edit`, `todo_write`, `job_list` y `job_kill` no se acogen al timeout de llamada de herramienta. `job_output` es dueño de su espera acotada porque un timeout de espera es un resultado exitoso de estado en vivo, no un fallo de herramienta.

Una futura herramienta grep/glob visible para el modelo puede implementarse sobre `ctx.shell` sin importar `@deepseek-ai/dsh-timeout`: reenvía `exec.signal` a `ctx.shell` y declara su propio `timeoutMs` (desde la configuración de su plugin) para que el aplicador lo aplique. Si el timeout del backend de bash-local llegara a ser un problema para esa herramienta, el seam de bash puede añadir más adelante un modo de deadline propiedad del llamador; eso es una decisión aparte.

## Alternativas consideradas

**Nombrar el plugin `tool-timeout`.** El nombre literal del Agent Note coincidía con el glob `packages/*/tool-*` del guard de completitud de `gen-tool-catalog`, que exige que cada coincidencia registre una herramienta visible para el modelo. Este plugin no registra ninguna — es un wrapper de `tools/execute` — así que un nombre `tool-*` o bien fallaría `verify-tool-catalog` o bien forzaría una entrada de arranque engañosa. El paquete es `@deepseek-ai/dsh-tool-call-timeout-policy` en lo que entonces era un grupo `timeout/` nuevo, desde entonces absorbido en `packages/guard/`; el `id` de cordis.yml puede seguir siendo `timeout-policy`.

**Conservar solo el manejo de timeout por herramienta.** Esa era la forma de `bash` y `web_fetch`, y coincide con Claude Code y Codex para comandos de shell. Pierde para las herramientas de estilo web porque cada nueva herramienta con timeout debe elegir validación, semántica de topes, docs, instantáneas y clasificación. El plugin centraliza la política y la clasificación dejando el schema de cada herramienta centrado en la entrada de negocio.

**Trasladar toda la política de timeout fuera de bash-local de inmediato.** Más limpio a largo plazo — bash-local se volvería un ejecutor de subprocesos puro y todos los llamadores serían dueños de sus deadlines. Pierde como primer paso porque los hooks llaman a `ctx.shell` directamente y la herramienta de modelo bash tiene semántica de primer plano/segundo plano que no es el mismo ciclo de vida de llamada de herramienta. Conservar `BASH_TIMEOUT` preserva esas vías mientras el timeout de llamada de herramienta se demuestra en herramientas más simples.

**Usar un presupuesto por defecto global para cada herramienta.** Cómodo, pero sorprende a los autores de herramientas: cualquier herramienta que accidentalmente tarde más que el presupuesto global empezaría a fallar en cuanto el plugin se cargara. Un presupuesto declarado por herramienta hace deliberada la adopción.

**Exponer una anulación `timeout_ms` visible para el modelo.** El `WebFetch`/`WebSearch` de Claude Code y las herramientas web de Codex mantienen el timeout fuera de la forma de la llamada del modelo. Una anulación del modelo convertiría el timeout en parte de la semántica del prompt y forzaría reglas de recorte de schema/argumentos dentro de `timeout-policy`. El timeout web sigue siendo solo política de despliegue.

**Dejar que `timeout-policy` compare argumentos de herramientas él mismo.** Un motor de reglas como «desactivar el timeout cuando `bash.run_in_background` sea verdadero» haría que el plugin de política conociera la semántica de argumentos específica de cada herramienta. Se evita no migrando bash al timeout de llamada de herramienta.

**Usar `tools/pre-execute` más `tools/post-execute` en lugar de un nuevo punto de extensión alrededor del despacho.** Un listener pre podría armar un deadline y mutar `exec.signal`; un listener post podría clasificar y sustituir. Pierde porque el ciclo de vida del deadline cruzaría dos waterfalls independientes: un mapa de id de llamada, limpieza en cada vía de pre-denegación/lanzamiento de herramienta/lanzamiento post/dispose, y reglas de orden con todos los demás listeners. `tools/pre-execute` también es la puerta de permitir/denegar, no un wrapper de ejecución. `tools/execute` le da al timeout un único ámbito léxico: armar, delegar, clasificar, liberar.

**Usar `Promise.race` para imponer timeouts en herramientas no cooperativas.** Rechazado por la misma razón que el Agent Note de la biblioteca de timeout: devuelve el control al llamador mientras el proceso, la captura o la operación del provider subyacentes pueden seguir en ejecución. El plugin solo envía una señal; la terminación sigue siendo responsabilidad de la implementación.

## Consecuencias

- `@deepseek-ai/dsh-tools` gana una superficie alrededor del despacho después de que los puntos de intercepción dividieran deliberadamente los hooks de herramienta pre/post. Su contrato es estrecho — envolver el despacho del registro, no sustituir la política de pre-puerta ni de post-resultado — y el `next()` base es despacho-con-normalización para que un wrapper nunca vea un lanzamiento crudo de herramienta.
- Varios listeners de `tools/execute` se componen por el orden ordinario de waterfall de Cordis: un listener que llama a `next()` envuelve a los listeners aguas abajo más el despacho; uno que regresa sin `next()` los cortocircuita. Un despliegue que combine timeout con un futuro wrapper de reintento/sandbox/métricas elige la semántica por orden de registro («el timeout cubre todo el reintento» frente a «el timeout cubre cada intento»).
- La adopción por declaración es un riesgo deliberado de mala configuración: una herramienta puede declarar un `timeoutMs` sin honrar `exec.signal`, y esa herramienta no se detendrá en el timeout. El registro espera a ese cuerpo que no alcanza el reposo en lugar de competir con él, mientras el contrato del plugin afirma que declarar un presupuesto significa cooperativo; las herramientas web demuestran el patrón en herramientas que ya reenvían la señal.
- Durante la transición `bash` y las herramientas web migradas usan vías de timeout distintas a propósito: `TOOL_TIMEOUT` es el presupuesto de llamada de herramienta visible para el modelo, mientras que `BASH_TIMEOUT` sigue siendo el timeout de backend de bash usado por bash y por los hooks.
