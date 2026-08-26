# @deepseek-ai/dsh-subagent-claude-code

[English](README.md) | Español

Este paquete registra un provider de subagente (subagent) de Claude Code con nombre de Profile cuyo nombre por defecto es `claude-code`. Cada ejecución aceptada invoca el Claude Agent SDK oficial en el workspace de la sesión delegante, deja que el SDK fijado seleccione su CLI de plataforma instalada, envía una tarea de texto autocontenida y devuelve, mediante el contrato de resultado compartido de [`dsh-subagent`](../subagent/README.es.md), o bien la respuesta final estricta o bien un diagnóstico de fallo seguro separado.

## Inicio y propiedad

`start(request)` acepta solo una secuencia no vacía de bloques de texto y deriva el cwd del hijo de la Session del parent. Crea un `AbortController` privado, llama al `query()` oficial del SDK y publica la ejecución solo después de que el hook `spawnClaudeCodeProcess` del SDK haya suministrado un handle de CLI en vivo propiedad de [`dsh-subprocess`](../../subprocess/subprocess/README.es.md). Un fallo o una cancelación anteriores a la publicación cierran la consulta, terminan cualquier árbol de procesos adquirido, esperan a que salga y rechazan `start()`.

El SDK recibe exactamente la tarea de texto concatenada. El provider itera el flujo completo de mensajes del SDK y acepta solo un mensaje `result` con `subtype: "success"`, `is_error: false` y un `result` no en blanco, seguido de la finalización normal del iterador. Todo fallo se mapea igualmente a `error`: los cuatro subtipos de error de Agent SDK 0.3.220 conservan su categoría exacta, un éxito marcado como error o en blanco se convierte en `invalid-success`, un resultado ausente en `missing-result`, un fallo de consulta no clasificado en `unknown` y una salida temprana de la CLI en `process-exit`. El diagnóstico nombra también la etapa actual `query-start`, `query-run`, `process` o `teardown` e incluye de forma independiente un código de salida y una señal observados. El provider no produce ni `max-tokens` ni `refusal`.

La cancelación local gana la carrera del resultado y se mapea a `aborted` sin diagnóstico de fallo. `dispose()` es idempotente: aborta la ejecución, pide que la consulta del SDK se cierre, invoca la escalada compartida de terminación del árbol de procesos y espera la salida de todo el árbol. El cierre ordenado del SDK expresa la intención del protocolo; el handle del subproceso sigue siendo la autoridad para la quiescencia del proceso. Los rechazos de arranque y de teardown exponen los mismos hechos fijos y seguros de etapa y de proceso a través de su mensaje de Error, mientras que el error original del producto o del host permanece en la cadena de causas interna y en el log de host del Provider. El fallo de resultado y el fallo independiente de teardown permanecen separados.

## Ajustes nativos e interacción

El provider omite deliberadamente la opción `settingSources` del SDK. El SDK oficial lee por tanto los ajustes normales de usuario, de proyecto y locales de Claude del host relativos al cwd de la Session del parent, incluidos el estado de cuenta nativo y la configuración del producto. El provider ni copia ni filtra esos archivos y no crea ni modifica el estado de inicio de sesión. El `permissionMode` seleccionado por el Profile es la única anulación a nivel de consulta: Claude Code sigue siendo dueño de sus ajustes y de su sandbox, mientras que el modo nativo seleccionado decide cómo maneja las comprobaciones de permiso esta consulta no supervisada.

Cada consulta fija `persistSession: false` y desactiva `AskUserQuestion`. Salvo en modo bypass, `canUseTool` deniega de inmediato las solicitudes que aún requieren aprobación humana. El modo plan coloca además `ExitPlanMode` en `disallowedTools` del SDK, de modo que los ajustes nativos no pueden preaprobar una transición de vuelta a la ejecución y el modelo debe devolver el plan completado como respuesta final. La elicitación MCP se rechaza, el diálogo de respaldo de refusal conocido se cancela y los tipos de diálogo no declarados usan el comportamiento de fallo sin diálogo del SDK. Estas decisiones nunca esperan a una interfaz de usuario. Cuando ambos hechos contribuyen a una ejecución fallida, `SubagentResult.diagnostic` contiene primero la línea de fallo estructurada y después la decisión de permiso segura más reciente; la frontera compartida del resultado limita el texto completo a 4096 bytes UTF-8. Las ejecuciones con éxito y las canceladas localmente no exponen ninguno de los dos hechos capturados.

## Capacidades y contexto

El provider no anuncia funcionalidades opcionales de tiempo de arranque e informa `inheritsParentContext: false`. Claude Code recibe la tarea de texto autocontenida y el cwd de la Session del parent, pero no la conversación del parent, la persona, el filtro de herramientas, la política de profundidad ni el contrato de salida estructurada. Cada ejecución tiene una consulta de SDK, un controlador de cancelación, un proceso de CLI y una sesión de producto no persistida independientes.

## Configuración

| Clave | Por defecto | Significado |
|---|---|---|
| `providerName` | `claude-code` | Nombre de registro no vacío en `ctx.subagents`; cada instancia montada necesita un valor único. |
| `env` | `{}` | Entorno explícito de SDK/CLI superpuesto sobre el entorno compartido del parent con las credenciales eliminadas. |
| `permissionMode` | `dontAsk` | Política de permiso nativa no interactiva fijada para cada ejecución de esta instancia de Provider. |
| `disposeGraceMs` | `3000` | Período de gracia finito positivo en milisegundos, no mayor que [`MAX_TIMER_DELAY_MS`](../../util/timeout/README.es.md), entre los niveles de terminación del dueño compartido del árbol de procesos; la disposición espera entonces la salida de todo el árbol. |

| Valor de `permissionMode` | Comportamiento nativo |
|---|---|
| `dontAsk` | Deniega las operaciones que no estén ya autorizadas en lugar de preguntar. |
| `acceptEdits` | Acepta las ediciones de archivos; cualquier solicitud de permiso restante la deniega el callback no supervisado. |
| `auto` | Deja que el clasificador nativo de Claude Code permita o deniegue las solicitudes de permiso. |
| `plan` | Ejecuta en el modo de planificación nativo, deniega la aprobación de ejecución y devuelve el plan completado como respuesta final. |
| `bypassPermissions` | Establece explícitamente la confirmación peligrosa del SDK y omite las comprobaciones de permiso. |

En producción se omite `pathToClaudeCodeExecutable`, así que Agent SDK 0.3.220 selecciona el `claude` o `claude.exe` nativo correspondiente de su propio paquete de plataforma y pasa ese comando absoluto a través del hook de spawn personalizado a `dsh-subprocess`. El provider no inspecciona `PATH`, no implementa la selección de plataforma ni recurre a un `claude` del host. Los ajustes y la autenticación nativos siguen siendo autoritativos, mientras que `permissionMode` es la única anulación de política a nivel de consulta. El plugin no selecciona un modelo, no crea un home de producto, no inicia sesión ni sondea una cuenta. Las variables ambientales con forma de credencial se eliminan antes de aplicar la superposición explícita de `env`, así que una clave de API o un token destinado al hijo debe suministrarse ahí. Las variables de endpoint que no son credenciales, como `ANTHROPIC_BASE_URL`, junto con los valores ambientales ordinarios como `PATH` y `HOME`, siguen heredándose salvo que se anulen; `PATH` no elige el ejecutable de Claude.

Este paquete es un Bundle de Profile opcional. Instálalo en el Profile de destino y reinicia después ese Profile; la instalación lleva el Agent SDK fijado y un payload de CLI de plataforma compatible a ese Profile, mientras que la capa declarada de `cordis.patch.yml` registra solo el provider de host `claude-code` inactivo y no arranca ningún proceso de Claude. Eliminar el paquete retira ese provider y su cierre de runtime privado en el siguiente arranque del Profile.

```sh
dsh plugin --profile <name> add @deepseek-ai/dsh-subagent-claude-code
dsh plugin --profile <name> remove @deepseek-ai/dsh-subagent-claude-code
dsh --profile <name>
```

La instalación controla la disponibilidad en el host, no el permiso de modelo. El Bundle suministra la fila por defecto inactiva `claude-code`; el Profile puede sustituir la configuración completa de esa fila o montar filas adicionales con valores distintos de `providerName`, `permissionMode` y `env`. Cargar una instancia no arranca ningún proceso de Claude hasta que una herramienta vinculada la llama. Cada fila de `dsh-tool-subagent` nombra un provider y necesita su propio `toolName`, así que el modelo ve herramientas estáticas en lugar de un selector dinámico de providers. Los Agent Presets completos llevan una fila de herramienta de producto por defecto correspondiente con `disabled: true`; copia un preset y elimina ese campo para exponer `subagent_claude_code` solo a los agents compuestos a partir de la copia. Su política `one-shot` mantiene en primer plano las llamadas `run_in_background` omitidas o en `false`, mientras que un `true` explícito devuelve un id de Job propiedad del parent para `job_output` o `job_kill`. El host base y los presets completos ya proporcionan el registro y los controles genéricos de Job.

La composición independiente siguiente muestra la capacidad explícita completa. Un Profile basado en `@deepseek-ai/dsh-base` conserva sus filas de Job existentes, añade las filas de provider y de herramienta del producto y no monta servicios de Job duplicados.

```yaml
- id: subagent-claude-safe
  name: '@deepseek-ai/dsh-subagent-claude-code'
  config:
    providerName: claude-safe
    permissionMode: dontAsk
    env:
      ANTHROPIC_API_KEY: !!js process.env.ANTHROPIC_API_KEY

- id: subagent-claude-bypass
  name: '@deepseek-ai/dsh-subagent-claude-code'
  config:
    providerName: claude-bypass
    permissionMode: bypassPermissions
    env:
      ANTHROPIC_API_KEY: !!js process.env.ANTHROPIC_API_KEY
```

```yaml
- id: jobs
  name: '@deepseek-ai/dsh-jobs-local'

- id: tool-jobs
  name: '@deepseek-ai/dsh-tool-jobs'

- id: tool-subagent-claude-safe
  name: '@deepseek-ai/dsh-tool-subagent'
  disabled: true
  config:
    provider: claude-safe
    toolName: subagent_claude_safe
    backgroundMode: one-shot
    maxDepth: provider-managed

- id: tool-subagent-claude-bypass
  name: '@deepseek-ai/dsh-tool-subagent'
  config:
    provider: claude-bypass
    toolName: subagent_claude_bypass
    backgroundMode: one-shot
    maxDepth: provider-managed
```

## Compatibilidad del producto y evidencia

La dependencia de runtime está fijada a `@anthropic-ai/claude-agent-sdk@0.3.220`, cuyos ocho paquetes de plataforma incluyen Claude Code 2.1.220. Una instalación normal selecciona un payload para el sistema operativo, la CPU y la libc de Linux actuales. Para el payload darwin-arm64 actual, `npm pack --dry-run --json` informa de 74,858,812 bytes empaquetados y 256,908,856 bytes desempaquetados; otras plataformas pueden diferir, y estos valores son divulgación y no un umbral de instalación. La prueba real del producto sin clave ejecuta la CLI seleccionada por el SDK contra un fixture (datos de prueba) de Messages en loopback y comprueba que el argv compartido del subproceso empieza por el ejecutable nativo de ese paquete de plataforma. La composición de Loader demuestra que instalar el Bundle registra solo el provider inactivo de Claude Code y no arranca ningún proceso del producto.

Instalar omitiendo las dependencias opcionales, usar una plataforma no soportada o perder el payload seleccionado deja inactivo el registro del provider, pero hace que la primera delegación falle en la frontera de arranque del SDK. El llamador recibe el hecho de fallo seguro `query-start` / `unknown`; el error del payload nativo permanece solo en la cadena de causas interna y en el log de host del Provider. El provider no sondea una CLI del host ni reintenta con una.

La composición de Loader demuestra que el valor por defecto del Bundle, dos instancias de Claude con nombre adicionales y el paquete Codex existente coexisten sin arrancar ninguno de los dos productos.

La autorización de distribución con ámbito de identidad del propietario del proyecto cubre el SDK oficial y los payloads oficiales de CLI/plataforma declarados por cada versión del SDK. [`THIRD_PARTY_NOTICES.md`](../../../THIRD_PARTY_NOTICES.md) divulga el cierre de payload opcional actual sin clasificar sus términos declarados como permisivos; las dependencias de runtime no permisivas no relacionadas siguen fallando la compuerta de notices.

## Model Experience

### Solicitud del hijo

#### Lo que ve el modelo

El hijo de Claude Code recibe la tarea de texto autocontenida como una consulta de SDK nueva. Su workspace es el cwd de la Session del parent; su modelo, sus instrucciones de sistema, sus herramientas, su sandbox y su autenticación provienen de los ajustes nativos de Claude, la configuración de Profile de la instancia de Provider seleccionada fija el entorno y el modo de permiso no interactivo de la consulta, y la versión del ejecutable proviene del payload de plataforma del SDK fijado por el Bundle.

#### Efecto de tokens

El hijo paga un contexto y una consulta independientes de Claude Code. Los tokens del hijo no entran en el contexto del parent.

#### Efecto de KV Cache

Independiente de la caché de solicitudes del parent. La reutilización depende solo del modelo, las instrucciones, las herramientas, los ajustes nativos y la consulta nueva de Claude Code.

### Programación y resultados del parent, indirectamente

#### Lo que ve el modelo

A través de `dsh-tool-subagent`, una llamada en primer plano da al parent la respuesta final estricta de Claude Code o un error que contiene la razón de parada y el diagnóstico seguro opcional de un resultado no completado. Ese diagnóstico puede distinguir la categoría fija de error del SDK, la etapa del ciclo de vida y el resultado de proceso observado sin copiar el texto bruto del producto. Una llamada en segundo plano devuelve primero un id de Job; los controles genéricos de job entregan después un aviso de finalización, exponen la misma respuesta final o el detalle de estado fallido a través de `job_output` y permiten que `job_kill` solicite la cancelación. El razonamiento de Claude Code, la actividad de herramientas, los mensajes intermedios, el stderr, los diffs de workspace, el uso, los ids de producto, las entradas de herramientas y los payloads de protocolo brutos no se copian en la Session del parent.

#### Efecto de tokens

La entrada en primer plano crece con la respuesta final o el error conservados. La entrada en segundo plano incluye además el acuse de inicio, el aviso de finalización y cualquier resultado posterior de `job_output`, `job_kill` o de estado; los tokens del hijo siguen sin entrar en el contexto del parent. Este provider no añade schema de herramienta al parent por sí mismo.

#### Efecto de KV Cache

Solo de añadido: el primer plano añade un resultado después del prefijo reutilizable del parent, mientras que el segundo plano añade el acuse del Job, el aviso y los resultados posteriores de control o recolección. La programación en segundo plano puede añadir un turno impulsado por avisos, pero ninguno de estos mensajes reescribe el prefijo anterior.

## Limitaciones conocidas y trabajo diferido

- **Una consulta y un proceso nuevos por ejecución** — no hay continuación, reanudación, pooling, flujo de progreso ni persistencia de sesión de producto.
- **Selección estática de instancias** — las filas de Profile fijan los nombres de provider y los enlaces de herramienta; las llamadas no pueden elegir un provider dinámicamente y toda herramienta expuesta necesita un `toolName` único.
- **Los ajustes del host son deliberadamente autoritativos** — los ajustes de proyecto y de usuario pueden cambiar el modelo, las herramientas y el comportamiento; el provider no ofrece un modo de producción filtrado o hermético.
- **La autenticación y el estado de la cuenta siguen siendo nativos** — el Bundle suministra la CLI pero no crea una cuenta, no inicia sesión ni reescribe los ajustes de Claude; los fallos de configuración y de autenticación afloran con su etapa del ciclo de vida y el respaldo seguro `unknown` en lugar de una clasificación pública separada.
- **El payload de plataforma del SDK se exige en el momento de la delegación** — las instalaciones que omiten dependencias opcionales, las plataformas no soportadas y los payloads ausentes o dañados fallan en la primera consulta; no hay respaldo de CLI del host.
- **Sin camino de interacción humana** — `AskUserQuestion` está desactivado, las solicitudes de permiso se deniegan, la elicitación MCP se rechaza y los diálogos bloqueantes fallan en modo cerrado en lugar de suspenderse.
- **El payload del asistente es solo texto final** — una ejecución fallida puede exponer además el diagnóstico seguro separado; el razonamiento, los mensajes intermedios, el tráfico de herramientas, el uso, el stderr y los diffs de workspace permanecen locales al producto, mientras que los ids, avisos y estados genéricos de Job provienen del runtime de jobs compartido.
- **Sin capacidades compartidas opcionales** — los schemas de salida, las personas de hijo, el filtrado de herramientas y la aplicación de profundidad del harness los rechaza el servicio compartido para este provider.
- **Sin tiempo de espera de reloj de pared ni reversión de efectos secundarios** — el llamador cancela el trabajo largo, y los archivos o sistemas externos modificados antes de la cancelación no se restauran.
