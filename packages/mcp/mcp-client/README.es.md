# @deepseek-ai/dsh-mcp-client

[English](README.md) | Español

Plugin puente de cliente MCP: se conecta a servidores externos de [Model Context Protocol](https://modelcontextprotocol.io/) y registra sus herramientas en `ctx.tools`, poniéndolas a disposición del modelo como herramientas nativas con nombres calificados por servidor (`mcp__<serverName>__<rawName>`).

## Uso

Una instancia de plugin por servidor MCP en `cordis.yml`:

```yaml
- id: mcp-github
  name: '@deepseek-ai/dsh-mcp-client'
  config:
    serverName: github
    transport: stdio
    command: npx
    args: ['-y', '@modelcontextprotocol/server-github']
    env:
      GITHUB_TOKEN: !!js process.env.GITHUB_TOKEN

- id: mcp-web
  name: '@deepseek-ai/dsh-mcp-client'
  config:
    serverName: web
    transport: streamable-http
    url: http://localhost:3000/mcp
    headers:
      Authorization: !!js '`Bearer ${process.env.MCP_TOKEN}`'
```

El modelo ve `mcp__github__create_issue`, `mcp__web__search`, … — la misma forma calificada por servidor que usan Claude Code y Codex. El HMR intercambia en caliente: editar la entrada dispara desconexión + reconexión sin reiniciar el proceso; un `serverName` sin cambios reproduce nombres de herramienta idénticos.

## Config

| Campo | Transporte | Obligatorio | Descripción |
|---|---|---|---|
| `transport` | ambos | sí | `"stdio"` o `"streamable-http"` |
| `serverName` | ambos | sí | Namespace para los nombres de herramienta orientados al modelo de este servidor; `[A-Za-z0-9_-]{1,32}`, único entre las instancias activas |
| `command` | stdio | sí | Ejecutable a lanzar (spawn) |
| `args` | stdio | no | Argumentos pasados al comando |
| `env` | stdio | no | Variables de entorno extra fusionadas sobre el entorno ambiental depurado |
| `cwd` | stdio | no | Directorio de trabajo del proceso hijo |
| `url` | http | sí | URL del servidor MCP |
| `headers` | http | no | Cabeceras extra (p. ej. tokens de autenticación) |
| `toolCallTimeoutMs` | ambos | no | Timeout por invocación de `callTool` (predeterminado 60000) |
| `failOnStartupError` | ambos | no | Rechaza la activación del plugin cuando falla la conexión inicial o la sincronización de herramientas (predeterminado `false`) |
| `reconnect.enabled` | ambos | no | Reconectar automáticamente tras una conexión perdida (predeterminado `true`) |
| `reconnect.initialDelayMs` | ambos | no | Primer retardo de reconexión en ms; se duplica por cada intento fallido consecutivo (predeterminado 500) |
| `reconnect.maxDelayMs` | ambos | no | Tope de backoff en ms; también el tiempo de actividad tras el cual se reinicia el presupuesto de intentos (predeterminado 30000) |
| `reconnect.maxAttempts` | ambos | no | Intentos fallidos consecutivos por interrupción antes de rendirse definitivamente (predeterminado 10) |

## Nombrado de herramientas

Cada herramienta MCP tiene dos nombres: el nombre MCP crudo (enviado por el wire en `tools/call`) y el nombre público `mcp__<serverName>__<rawName>` registrado en `ctx.tools`. Los nombres públicos se normalizan al contrato de nombres de función de DeepSeek (64 caracteres, `[A-Za-z0-9_-]`); cuando el reemplazo o el truncamiento cambia el nombre, se añade un hash determinista de 12 caracteres hex de `(serverName, rawName)` para que herramientas distintas nunca colapsen en un solo nombre. Los nombres son funciones puras de `(serverName, rawName)` — el orden de conexión, las re-sincronizaciones y otros servidores nunca renombran una herramienta.

- Dos servidores que publican el mismo nombre crudo (p. ej. `search`) conviven bajo sus namespaces.
- Un `serverName` duplicado entre instancias activas hace fallar la última instancia de plugin en la carga.
- Un servidor que lista el mismo nombre de herramienta dos veces se rechaza como lista de herramientas inválida.
- Un registro externo que usurpa el namespace de este servidor revierte toda la generación (nunca un conjunto parcial), con un error bien visible.

## Comportamiento

- Al conectar: la activación del plugin espera a `listTools()` y registra cada herramienta mediante `ctx.tools.register()` bajo su nombre público antes de que la composición inicie su primer turno. El fallo de conexión inicial, descubrimiento o registro siempre se registra en el log; rechaza la activación cuando `failOnStartupError` es true y, en caso contrario, se activa sin herramientas.
- Escucha `notifications/tools/list_changed` → re-sincroniza; un fallo en la fase de obtención mantiene registrada la generación anterior, mientras que un conflicto de registro revierte la generación intentada y no deja herramientas de ese servidor.
- Ejecución de herramienta: `client.callTool({ name: rawName, arguments }, { signal })` con soporte de timeout y abort: el nombre público nunca se envía al servidor.
- El éxito canónico es `{ content: JsonValue[], structuredContent? }`; los bloques JSON completos de MCP sobreviven para los llamadores programáticos. Un `outputSchema` anunciado y admitido valida `structuredContent`; el vocabulario de schema no admitido cae en `JsonValue` sin restricciones.
- El renderizado Native/modelo conserva el orden de los bloques MCP. Los tramos de tipo texto se unen con saltos de línea; los enlaces de recursos conservan su nombre y URI como texto; las imágenes admitidas se convierten en bloques de imagen durables del core solo cuando `ctx.attachments` está montado y la ruta exacta del modelo llamador declara explícitamente entrada de imágenes. Todo el lote de imágenes se decodifica y admite antes de guardar cualquiera de sus miembros. Un lote de imágenes malformado o rechazado, el audio, los recursos embebidos y los bloques no admitidos se convierten en texto diagnóstico explícito en lugar de desaparecer.
- Ante desconexión/caída: el supervisor reinicia la configuración original del servidor con backoff exponencial (`reconnect.initialDelayMs` duplicándose hasta `reconnect.maxDelayMs`) y vuelve a ejecutar el descubrimiento al tener éxito: la generación recuperada reemplaza a la anterior, así las herramientas ni se duplican ni se filtran. Durante la interrupción, la última generación buena sigue registrada; las llamadas contra ella fallan hasta la recuperación.
- La reconexión se presupuesta por interrupción: tras `reconnect.maxAttempts` fallos consecutivos, las herramientas del servidor se dan de baja y la reconexión se detiene hasta una recarga por HMR o un reinicio del Host. Una conexión que sobrevive más allá de `maxDelayMs` reinicia el presupuesto, así que un servidor que cae ocasionalmente se recupera indefinidamente, mientras que uno en bucle de caídas — incluso con conexiones brevemente exitosas — agota el tope en lugar de reiniciarse para siempre.
- Los estados de reconexión son visibles para el usuario en los logs: reconectando (warn, con el número de intento y el retardo), recuperado (info), fallo final y pérdida deshabilitada (error). La disposición cancela cualquier reconexión pendiente. Con `reconnect.enabled: false`, una conexión perdida mantiene las herramientas registradas pero fallando hasta una recarga: el comportamiento de recuperación manual.

## Servicios consumidos

| Servicio | Uso |
|---|---|
| `ctx.tools` | Registrar/dar de baja herramientas MCP |
| `ctx.attachments` | Validar y persistir opcionalmente los lotes de resultados de imagen antes de la proyección al modelo |
| `ctx.llm` | Probar opcionalmente que la ruta exacta del llamador admite explícitamente entrada de imágenes |

## Experiencia del modelo

### Herramientas MCP descubiertas

#### Qué ve el modelo

Tras el éxito del descubrimiento inicial, cada herramienta MCP anunciada aparece como herramienta nativa llamada `mcp__<serverName>__<rawName>` (o su forma normalizada determinista), con la descripción y el schema de entrada proporcionados por el servidor. Una re-sincronización exitosa — incluida la posterior a una reconexión automática — reemplaza la generación; la disposición del plugin o un presupuesto de reconexión agotado la elimina.

#### Efecto de tokens

El coste de schema dependiente de los datos se paga en cada solicitud mientras las herramientas estén registradas. La re-sincronización reemplaza los schemas en lugar de acumularlos, y el nombre calificado por servidor añade tokens a cada definición y llamada de herramienta.

#### Efecto de KV Cache

Estable en el prefijo mientras el conjunto de herramientas descubierto y los schemas no cambien. Una re-sincronización que añade, elimina, renombra o cambia una herramienta reemplaza las definiciones y puede invalidar la reutilización desde el primer token de schema que cambie; una reconexión que recupera una lista sin cambios reproduce definiciones idénticas y se mantiene estable en el prefijo.

### Historial y resultados de llamadas de herramienta

#### Qué ve el modelo

El nombre público de la herramienta y los argumentos JSON permanecen en el historial del asistente. El valor canónico local de la ejecución conserva siempre los bloques JSON completos de MCP y el contenido estructurado opcional para los llamadores programáticos y de Code Mode. En contexto Native, los bloques de imagen admitidos se proyectan de forma durable junto al texto en su orden original tras probar la capacidad exacta de la ruta; Code Mode transporta además esa proyección enriquecida asentada a través del resultado externo de `run_code` sin cambiar el valor canónico vinculado. Las imágenes rechazadas, el audio, los recursos embebidos, los enlaces de recursos y los bloques desconocidos siguen visibles como diagnósticos de texto acotados, y `isError` de MCP rechaza la llamada antes de la persistencia de imágenes.

#### Efecto de tokens

Los argumentos, el texto mapeado y las referencias durables de imagen se retienen hasta la compactación. El base64 MCP en línea queda solo en el valor canónico local de la ejecución y nunca se copia en un evento de sesión; el provider lee los bytes verificados del almacén de attachments. Los payloads de audio y de recursos embebidos se mantienen fuera del contexto del modelo.

#### Efecto de KV Cache

Solo adición; el contenido recién visible sigue el prefijo de solicitud reutilizable y no invalida las entradas de KV cache existentes.

## Limitaciones conocidas y trabajo diferido

- **Las herramientas son la única capacidad MCP puenteada** — Resources y Prompts no tienen Consumer en el harness y quedan diferidos.
- **El timeout de arranque se hereda del SDK de MCP** — DSH aún no expone un timeout de conexión/descubrimiento. Cada solicitud de initialize o de `tools/list` paginado usa el predeterminado de 60 segundos del SDK, así que un servidor que no responde o una cadena de cursores puede retrasar tanto la activación como el cierre mientras se asienta la sincronización inicial.
- **La reconexión se dispara al cerrarse el transporte** — un hijo stdio que cae la dispara; los fallos de Streamable HTTP afloran por solicitud y a través de la propia recuperación del stream SSE del transporte del SDK, así que un servidor HTTP inalcanzable se reintenta por llamada en lugar de ser relanzado por el supervisor.
- **La imagen es el único puente durable de resultados enriquecidos** — PNG, JPEG, WebP y GIF pueden entrar en el contexto Native tras probar la capacidad exacta. Los payloads de audio y de recursos embebidos permanecen locales a la ejecución con diagnósticos explícitos, mientras que los enlaces de recursos conservan solo su nombre y URI como texto.
- **Los schemas de salida MCP no admitidos no se aplican** — `structuredContent` cae en `JsonValue` cuando el schema anunciado usa vocabulario fuera del subconjunto del harness.
