# @deepseek-ai/dsh-subagent-codex

[English](README.md) | Español

Este paquete registra un provider de subagente (subagent) de Codex con nombre de Profile cuyo nombre por defecto es `codex`. Cada ejecución aceptada arranca el wrapper oficial de Codex del paquete con `app-server --stdio` en el workspace de la sesión delegante, crea un thread efímero de Codex, envía una tarea de texto autocontenida y devuelve, mediante el contrato de resultado compartido de [`dsh-subagent`](../subagent/README.es.md), o bien la respuesta final seleccionada o bien un diagnóstico de fallo seguro separado.

## Inicio y propiedad

`start(request)` acepta solo una secuencia no vacía de bloques de texto y deriva el cwd del hijo de la Session del parent. Hace entonces spawn del comando fijo a través de [`dsh-subprocess`](../../subprocess/subprocess/README.es.md), realiza `initialize` → `initialized`, mapea el modo seleccionado por el Profile a los campos oficiales de aprobación/revisor/sandbox de `thread/start` junto a `{ cwd, ephemeral: true }`, y publica la ejecución solo después de que Codex devuelva un thread efímero válido. Un fallo o una cancelación anteriores a la publicación cierran el cableado, terminan el árbol de procesos gestionado, esperan a que salga y rechazan `start()`. Los rechazos que no son cancelación exponen solo la etapa fija `initialize` o `thread-start` más un resultado de proceso ya observado; los errores brutos del producto y del host permanecen en las cadenas de causas internas.

El `run.result` publicado arranca exactamente un turno. Acepta solo notificaciones del thread y del turno de esa ejecución y espera después la notificación terminal autoritativa `turn/completed`. Gana el `agentMessage` más reciente con `phase: "final_answer"`; cuando Codex no emite una fase final explícita, el mensaje más reciente con `phase: null` es el respaldo de compatibilidad. El comentario nunca sustituye a ninguna de las dos respuestas, y un turno con éxito sin respuesta no en blanco se liquida como error.

Para las aprobaciones de comando y de archivo, el provider no supervisado selecciona una decisión de no aprobación ofrecida por la solicitud, prefiriendo `cancel`; la forma estable de solicitud 0.147.0 sin lista de decisiones ofrecidas recurre a `decline`. Responde a las solicitudes de permiso con un conjunto de permisos vacío con ámbito de turno, responde a las solicitudes de entrada de usuario sin respuestas y rechaza la elicitación MCP. Una solicitud sin respuesta no supervisada legal, o cualquier solicitud de servidor desconocida, hace fallar la ejecución. El cableado registra solo el modo efectivo, la categoría de solicitud, la decisión y la razón segura fija. Reconoce además los elementos de comando/archivo rechazados y los terminales `sandboxError`. Codex 0.147.0 escribe algunos rechazos tempranos de `never` y violaciones de sandbox solo en el stderr estructurado, así que el Provider canaliza el stderr, lo reenvía sin cambios al host y busca dos firmas fijas en una cola acotada por ejecución; el stderr bruto nunca entra en el diagnóstico.

La cancelación local gana la carrera del resultado y se mapea a `aborted`. Para los turnos fallidos, el diagnóstico conserva las once variantes de cadena y las cinco de objeto de la unión `codexErrorInfo` de Codex 0.147.0; las cuatro variantes de conexión/flujo conservan un `httpStatusCode` numérico cuando se suministra, mientras que `activeTurnNotSteerable` no expone `turnKind`. El diagnóstico nombra además `turn-start`, `turn` o `process`, incluye de forma independiente el código de salida y la señal disponibles, y usa `unknown` para los valores no reconocidos o malformados sin copiar campos brutos. `contextWindowExceeded` sigue siendo `max-tokens`; cualquier otra interrupción o fallo remoto sigue siendo `error`, y el provider no produce `refusal`. Una decisión de permiso que contribuye sigue a la línea de fallo estructurada. Las ejecuciones con éxito y las canceladas localmente omiten ambos hechos.

`dispose()` es idempotente: solicita un `turn/interrupt` de mejor esfuerzo con ambos ids actuales cuando se conocen, cierra el cableado JSON-RPC, termina stdin, invoca la escalada compartida de terminación del árbol de procesos, espera la salida de todo el árbol y desconecta el observador de stderr. Un rechazo independiente de limpieza usa la etapa fija `teardown` y cualquier resultado de proceso disponible. Cuando fallan el arranque y la reversión, el mensaje agregado de nivel superior conserva ambas líneas de etapa seguras mientras los fallos brutos permanecen internos.

## Capacidades y contexto

El provider no anuncia funcionalidades opcionales de tiempo de arranque e informa `inheritsParentContext: false`. Codex recibe la tarea de texto autocontenida y el cwd de la Session del parent, pero no la conversación del parent, la persona, el filtro de herramientas, la política de profundidad ni el contrato de salida estructurada. El id de thread efímero y el id de turno de Codex permanecen privados a esta ejecución y nunca se persisten en la Session del parent.

## Configuración

| Clave | Por defecto | Significado |
|---|---|---|
| `providerName` | `codex` | Nombre de registro no vacío en `ctx.subagents`; cada instancia montada necesita un valor único. |
| `env` | `{}` | Entorno explícito del hijo superpuesto sobre el entorno del parent del seam de subproceso con las credenciales eliminadas. |
| `permissionMode` | `never` | Modo nativo no interactivo de aprobación y sandbox fijado para cada thread de esta instancia de Provider. |
| `disposeGraceMs` | `3000` | Período de gracia finito positivo en milisegundos, no mayor que [`MAX_TIMER_DELAY_MS`](../../util/timeout/README.es.md), entre los niveles de terminación del dueño compartido del árbol de procesos; la disposición espera entonces la salida de todo el árbol. |

| Valor de `permissionMode` | Campos de `thread/start` | Comportamiento nativo |
|---|---|---|
| `never` | `approvalPolicy: never`; sandbox omitido | Nunca pide aprobación; los fallos de ejecución vuelven al modelo bajo el sandbox nativo. |
| `approve-for-me` | `approvalPolicy: on-request`, `approvalsReviewer: auto_review`, `sandbox: workspace-write` | Enruta las solicitudes de permiso por la revisión automática de Codex sin intervención humana. |
| `dangerously-bypass-approvals-and-sandbox` | `approvalPolicy: never`, `sandbox: danger-full-access` | Omite la aplicación de la aprobación y del sandbox; este valor debe seleccionarse explícitamente. |

En producción se resuelve el bin `codex` declarado por su dependencia fijada `@openai/codex@0.147.0` y se lanza ese wrapper de JavaScript con el ejecutable de Node actual. El wrapper selecciona el payload de plataforma nativo correspondiente; el provider no inspecciona ni usa como respaldo un `codex` del host en `PATH`. La configuración y la autenticación nativas de Codex siguen siendo autoritativas a través del cwd del parent, `HOME` y `CODEX_HOME`, mientras que el Provider anula solo los campos seleccionados de aprobación/revisor/sandbox del thread. Todos los demás ajustes de proyecto, modelo, provider, MCP, hook, skill y cuenta siguen siendo nativos. El plugin no selecciona un modelo, no crea `CODEX_HOME`, no inicia sesión ni sondea una cuenta. Las variables ambientales con forma de credencial las elimina el seam de subproceso antes de aplicar la superposición explícita de `env`.

Este paquete es un Bundle de Profile opcional. Instálalo en el Profile de destino y reinicia después ese Profile; la instalación lleva el wrapper oficial y un payload de plataforma nativo compatible a ese Profile, mientras que la capa declarada de `cordis.patch.yml` registra solo el provider de host `codex` inactivo y no arranca ningún proceso de Codex. Eliminar el paquete retira ese provider y su cierre de runtime privado en el siguiente arranque del Profile.

```sh
dsh plugin --profile <name> add @deepseek-ai/dsh-subagent-codex
dsh plugin --profile <name> remove @deepseek-ai/dsh-subagent-codex
dsh --profile <name>
```

La instalación controla la disponibilidad en el host, no el permiso de modelo. El Bundle suministra la fila por defecto inactiva `codex`; el Profile puede sustituir la configuración completa de esa fila o montar filas adicionales con valores distintos de `providerName`, `permissionMode` y `env`. Cargar una instancia no arranca ningún proceso de Codex hasta que una herramienta vinculada la llama. Cada fila de `dsh-tool-subagent` nombra un provider y necesita su propio `toolName`, así que el modelo ve herramientas estáticas en lugar de un selector dinámico de providers. Los Agent Presets completos llevan una fila de herramienta de producto por defecto correspondiente con `disabled: true`; copia un preset y elimina ese campo para exponer `subagent_codex` solo a los agents compuestos a partir de la copia. Su política `one-shot` mantiene en primer plano las llamadas `run_in_background` omitidas o en `false`, mientras que un `true` explícito devuelve un id de Job propiedad del parent para `job_output` o `job_kill`. El host base y los presets completos ya proporcionan el registro y los controles genéricos de Job.

La composición independiente siguiente muestra la capacidad explícita completa. Un Profile basado en `@deepseek-ai/dsh-base` conserva sus filas de Job existentes, añade las filas de provider y de herramienta del producto y no monta servicios de Job duplicados.

```yaml
- id: subagent-codex-safe
  name: '@deepseek-ai/dsh-subagent-codex'
  config:
    providerName: codex-safe
    permissionMode: never
    env:
      OPENAI_API_KEY: !!js process.env.OPENAI_API_KEY

- id: subagent-codex-bypass
  name: '@deepseek-ai/dsh-subagent-codex'
  config:
    providerName: codex-bypass
    permissionMode: dangerously-bypass-approvals-and-sandbox
    env:
      OPENAI_API_KEY: !!js process.env.OPENAI_API_KEY
```

```yaml
- id: jobs
  name: '@deepseek-ai/dsh-jobs-local'

- id: tool-jobs
  name: '@deepseek-ai/dsh-tool-jobs'

- id: tool-subagent-codex-safe
  name: '@deepseek-ai/dsh-tool-subagent'
  disabled: true
  config:
    provider: codex-safe
    toolName: subagent_codex_safe
    backgroundMode: one-shot
    maxDepth: provider-managed

- id: tool-subagent-codex-bypass
  name: '@deepseek-ai/dsh-tool-subagent'
  config:
    provider: codex-bypass
    toolName: subagent_codex_bypass
    backgroundMode: one-shot
    maxDepth: provider-managed
```

## Compatibilidad del producto y evidencia

El cableado de producción implementa deliberadamente solo los métodos de app-server que exige este contrato one-shot. La dependencia de runtime y los seis alias de dependencia opcional están fijados a `@openai/codex@0.147.0` / `codex-cli 0.147.0`. Una instalación normal selecciona un payload para el sistema operativo y la CPU actuales. Para el payload darwin-arm64 actual, `npm pack --dry-run --json @openai/codex@0.147.0-darwin-arm64` informa de 111,199,052 bytes empaquetados y 274,777,843 bytes desempaquetados. Ese paquete contiene recursos nativos de `codex`, `codex-code-mode-host`, `rg` y `zsh`; otras plataformas pueden diferir, y estos valores son divulgación y no un umbral de instalación.

La evidencia de schema generada y los tests del paquete fijan las dieciséis variantes de error-info, las ubicaciones de estado HTTP, las seis etapas del ciclo de vida, los resultados de proceso, el mapeo de razones de parada, el respaldo `unknown`, la sanitización, el orden de permisos, la cancelación, la concurrencia y la agregación de limpieza. La prueba real del producto sin clave conduce el wrapper del paquete contra un fixture (datos de prueba) de Responses en loopback y observa el argv local del paquete, la clave Bearer exacta, la tarea original, la respuesta final exacta byte a byte, el `never` a nivel de thread que anula el `on-request` ambiental, el arranque de revisión automática, el rechazo no supervisado sin efectos secundarios en archivos, un `internalServerError` real, el bypass peligroso explícito escribiendo en el almacenamiento temporal propiedad de la suite, el fallo de proceso/protocolo con hechos de salida seguros y la quiescencia del wrapper/nativo. El mismo nivel demuestra que dos instancias con nombre conservan entornos y modos nativos separados.

Instalar omitiendo las dependencias opcionales, usar una plataforma no soportada o perder el payload seleccionado hace que la primera delegación falle en `initialize` con la categoría segura `unknown` y cualquier resultado de proceso observado. El texto bruto del wrapper permanece en el stderr del host; el provider no sondea una CLI del host ni reintenta con una. Un fixture (datos de prueba) de wrapper aislado demuestra por separado el fallo del payload nativo y la ausencia de respaldo del host.

## Model Experience

### Solicitud del hijo

#### Lo que ve el modelo

El hijo de Codex recibe los bloques de texto autocontenidos como un turno en un thread efímero nuevo. Su workspace es el cwd de la Session del parent; su modelo, sus instrucciones de sistema, sus herramientas y su autenticación provienen de la configuración nativa de Codex, la configuración de Profile de la instancia de Provider seleccionada fija el entorno del thread, la política de aprobación no interactiva y el modo de sandbox, y la versión del ejecutable proviene del payload de plataforma fijado por el Bundle.

#### Efecto de tokens

El hijo paga un contexto y un turno independientes de Codex. Los tokens del hijo no entran en el contexto del parent.

#### Efecto de KV Cache

Independiente de la caché de solicitudes del parent. La reutilización depende solo del provider, el modelo, las instrucciones, las herramientas y la solicitud de thread efímero de Codex.

### Programación y resultados del parent, indirectamente

#### Lo que ve el modelo

A través de `dsh-tool-subagent`, una llamada en primer plano da al parent la respuesta final seleccionada de Codex o un error que contiene la razón de parada y el diagnóstico seguro opcional de un resultado no completado. El diagnóstico puede distinguir la categoría fija de error-info, la etapa de protocolo, el estado HTTP numérico y el resultado de proceso observado sin copiar la prosa del producto. Una llamada en segundo plano devuelve primero un id de Job; los controles genéricos de job entregan después un aviso de finalización, exponen la misma respuesta final o el detalle de estado fallido a través de `job_output` y permiten que `job_kill` solicite la cancelación. El comentario, el razonamiento, la actividad de herramientas, el stderr bruto, los diffs de workspace, el uso, los ids de producto, los comandos, las rutas y los payloads de protocolo de Codex no se copian en la Session del parent.

#### Efecto de tokens

La entrada en primer plano crece con la respuesta final o el error conservados. La entrada en segundo plano incluye además el acuse de inicio, el aviso de finalización y cualquier resultado posterior de `job_output`, `job_kill` o de estado; los tokens del hijo siguen sin entrar en el contexto del parent. Este provider no añade schema de herramienta al parent por sí mismo.

#### Efecto de KV Cache

Solo de añadido: el primer plano añade un resultado después del prefijo reutilizable del parent, mientras que el segundo plano añade el acuse del Job, el aviso y los resultados posteriores de control o recolección. La programación en segundo plano puede añadir un turno impulsado por avisos, pero ninguno de estos mensajes reescribe el prefijo anterior.

## Limitaciones conocidas y trabajo diferido

- **Un proceso, un thread y un turno nuevos por ejecución** — no hay continuación, reanudación, pooling, flujo de progreso ni persistencia de sesión de producto.
- **Selección estática de instancias** — las filas de Profile fijan los nombres de provider y los enlaces de herramienta; las llamadas no pueden elegir un provider dinámicamente y toda herramienta expuesta necesita un `toolName` único.
- **La autenticación y el estado de la cuenta siguen siendo nativos** — el Bundle suministra la CLI pero no crea una cuenta, no inicia sesión, no confía en un proyecto ni reescribe los ajustes de Codex; los fallos de configuración y de autenticación afloran con su etapa del ciclo de vida y el respaldo seguro `unknown` en lugar de una taxonomía pública separada.
- **El payload de plataforma nativo se exige en el momento de la delegación** — las instalaciones que omiten dependencias opcionales, las plataformas no soportadas y los payloads ausentes o dañados fallan en la primera ejecución; no hay respaldo de CLI del host.
- **La compatibilidad está fijada por la evidencia de desarrollo** — actualizar desde la línea base de protocolo 0.147.0 verificada exige regenerar la evidencia de schema upstream y volver a ejecutar los tests de handshake, selección de respuesta, aprobación, cancelación, producto real sin clave y nonce DeepSeek con credenciales.
- **Sin camino de aprobación humana** — las solicitudes de aprobación no supervisadas conocidas se deniegan y las solicitudes de servidor desconocidas fallan en modo cerrado; los tres modos de Profile nunca crean un canal de interacción DSH ni una política de permiso por llamada.
- **El payload del asistente es solo texto final** — una ejecución fallida puede exponer además el diagnóstico seguro separado; el razonamiento, el comentario, los mensajes intermedios, el tráfico de herramientas, el uso, el stderr bruto y los diffs de workspace permanecen fuera de la Session del parent, mientras que los ids, avisos y estados genéricos de Job provienen del runtime de jobs compartido.
- **Sin capacidades compartidas opcionales** — los schemas de salida, las personas de hijo, el filtrado de herramientas y la aplicación de profundidad del harness los rechaza el servicio compartido para este provider.
- **Sin tiempo de espera de reloj de pared ni reversión de efectos secundarios** — el llamador cancela el trabajo largo, y los archivos o sistemas externos modificados antes de la cancelación no se restauran.
