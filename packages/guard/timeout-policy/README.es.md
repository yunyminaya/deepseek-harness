# dsh-tool-call-timeout-policy

[English](README.md) | Español

Aplicador de tiempos de espera de llamadas de herramienta: un único listener de `tools/execute` en torno al despacho que arma un plazo cooperativo por llamada en `exec.signal` para una herramienta que declare `timeoutMs` en su `ToolDefinition` y devuelve un resultado estructurado `TOOL_TIMEOUT` cuando ese plazo gana. El presupuesto se lee de la propia declaración de la herramienta (`ToolDefinition.timeoutMs`, fijado por el plugin propietario de la herramienta), así que este plugin es **de configuración cero**. Es el wrapper de referencia de `tools/execute` y el hogar de aplicación de los presupuestos de llamadas de herramienta expuestos al modelo ([el Agent Note timeout-library](../../../.agents/notes/implemented/architecture/2026-07-06-timeout-deadline-library.md)).

## Plugin (namespace: `timeout-policy`)

Un plugin de función/namespace (`name` / `inject` / `apply`), no un servicio. No registra ninguna herramienta y no acepta configuración — consume el waterfall de `tools/execute` de `ctx.tools` (que el registro `dsh-tools` proporciona siempre) y lee el `timeoutMs` declarado de cada herramienta despachada desde el registro (`ctx.tools.get(exec.name)`).

```yaml
- id: timeout-policy
  name: '@deepseek-ai/dsh-tool-call-timeout-policy'
```

El presupuesto por herramienta lo declara el plugin de la herramienta (p. ej. la config `fetchTimeoutMs`/`searchTimeoutMs` de `dsh-tool-web`, adjunta como `ToolDefinition.timeoutMs`); este plugin solo lo aplica, así que un nombre de herramienta mal escrito no es posible.

### Comportamiento

Para una herramienta que **declare un `timeoutMs`**, el listener:

1. Lee el presupuesto de la propia declaración de la herramienta en el registro (`ctx.tools.get(exec.name)?.timeoutMs`) y arma `deadline(exec.signal, timeoutMs, 'TOOL_TIMEOUT')` — una sola señal que fusiona el abort del llamador con el temporizador de este plugin (`@deepseek-ai/dsh-timeout`).
2. Intercambia esa señal derivada en `exec` para el despacho aguas abajo y luego restaura la señal propia del llamador (el `next()` de cordis ignora los argumentos pasados, así que el wrapper muta el `exec` compartido in situ; la restauración hace que `tools/post-execute` siga viendo la señal del llamador).
3. Tras el despacho, si `timeoutOf(d.signal, 'TOOL_TIMEOUT')` coincide — el temporizador propio de este plugin se disparó — reemplaza el resultado por un resultado de herramienta estructurado `TOOL_TIMEOUT`: `{ isError: true, error: { message, info: { name: 'ToolTimeoutError', code: 'TOOL_TIMEOUT' } }, content: 'Error: tool call timed out after <ms>ms' }`.

Una herramienta que **no declare presupuesto** se delega intacta (sin plazo).

El `next()` base de `tools/execute` es el thunk de despacho-con-normalización del registro, así que cuando la señal de tiempo de espera llega a un provider que lanza su propio error de abort aguas arriba, el despacho lo convierte primero en un resultado de error normal, y este wrapper lo reemplaza después por `TOOL_TIMEOUT`. Ese orden es la razón por la que el reemplazo se fija en la señal (`timeoutOf`), no en la forma del resultado despachado.

### Cooperativo, no una interrupción forzosa

La señal derivada solo **notifica**; la terminación queda en manos de la herramienta y de la capacidad a la que esta reenvía `exec.signal` (la librería `dsh-timeout` no posee ningún kill). **Declarar `timeoutMs` significa por tanto «cooperativo con `exec.signal`»**: una herramienta que ignore la señal no se detendrá al agotarse el tiempo. Solo las herramientas que reenvían la señal deberían declararlo — las `web_fetch`/`web_search` incluidas (que reenvían a través de `ctx.web` a los providers) son la referencia. `TOOL_TIMEOUT` no necesita ningún evento de sesión para ser reconstruible: es el `tool/result` final expuesto al modelo, ya registrado por el bucle.

### Composición con otros wrappers de `tools/execute`

Varios listeners de `tools/execute` se componen según el orden de registro de cordis. Combinados con un futuro wrapper de reintento/sandbox/métricas, el orden de registro elige la semántica — «el tiempo de espera cubre toda la operación de reintento» (timeout registrado en el exterior) frente a «el tiempo de espera cubre cada intento» (timeout registrado en el interior).

## Model Experience

### Resultado de herramienta condicional

#### Lo que ve el modelo

Este plugin no añade prompt ni schema. Si gana un plazo declarado, reemplaza el resultado del provider por `Error: tool call timed out after <ms>ms` más el `TOOL_TIMEOUT` estructurado; si no, el resultado original pasa sin cambios.

#### Efecto de tokens

Cero tokens en las llamadas sin tiempo de espera. Un tiempo de espera añade un pequeño resultado de error retenido y puede evitar que un resultado tardío mayor del provider entre en el contexto.

#### Efecto de KV Cache

Solo añadidura; el contenido recién visible sigue el prefijo de petición reutilizable y no invalida las entradas de caché KV existentes.

## Limitaciones conocidas y trabajo diferido

- **Cooperativo, nunca una interrupción forzosa** — el plazo solo notifica mediante `exec.signal`; una herramienta que ignore la señal no se detiene al agotarse el tiempo (véase § Cooperativo, no una interrupción forzosa).
- **Sin presupuesto global** — solo las herramientas que declaren `timeoutMs` en su `ToolDefinition` reciben un plazo; no hay ningún valor por defecto a nivel de registro para las herramientas no declaradas (las `bash`/`read`/`write`/`edit` incluidas declaran deliberadamente ninguno).
