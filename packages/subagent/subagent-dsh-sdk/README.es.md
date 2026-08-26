# @deepseek-ai/dsh-subagent-dsh-sdk

[English](README.md) | Español

El provider SDK ejecuta cada subagente (subagent) como un runtime completo de DeepSeek Harness en un subproceso nuevo, conducido sobre JSON-RPC stdio a través del [cliente TypeScript del SDK](../../sdk/client/README.es.md). Es el segundo backend fuera de proceso junto a [`subagent-acp`](../subagent-acp/README.es.md), y difiere en el cableado y en el contrato del hijo: el backend ACP conduce cualquier agent del Agent Client Protocol; este backend conduce específicamente un runtime SDK del harness (el bin `dsh-jsonrpc-agent` o un ejecutable empaquetado), así que el hijo es un harness par completo —composición decidida por su propio `cordis.yml`, persistencia de sesión, ruta de modelo y herramientas—.

## Inicio y propiedad

`start(request)` resuelve el directorio de trabajo del hijo, hace spawn del runtime a través de `DeepSeekHarness` y completa el handshake de `initialize` (con la ruta `provider`/`model` configurada y el tope de salida opcional `maxTokens`) antes de cumplirse. El cumplimiento significa por tanto que el runtime del hijo está listo y que la propiedad ha pasado al llamador. Un fallo de spawn, de handshake o de cancelación anterior a la publicación solo rechaza una vez recolectado el subproceso; un fallo de resolución del directorio de trabajo rechaza antes de hacer spawn de nada.

El directorio de trabajo se resuelve exactamente como en el backend ACP, a través de los helpers compartidos fuera de proceso del seam ([`dsh-subagent`](../subagent/README.es.md)): la anulación `cwd` configurada cuando está fijada (validada una vez al cargar), y si no el cwd de la sesión parent delegante —nunca el cwd propio del proceso del servidor—. La ruta resuelta se convierte en el cwd del proceso hijo y en el cwd de workspace de su sesión SDK.

El id de ejecución devuelto se acuña en el namespace del parent; el id de sesión del runtime hijo existe solo dentro del proceso hijo. Tras la publicación, el provider posee una actividad SDK y lee la respuesta del hijo de sus eventos de sesión: el último `assistant/message` completo no vacío (se omite un mensaje de contenido vacío que registra uso), o el flujo `text-delta` acumulado cuando no existe tal mensaje. La salida parcial sigue disponible tras una cancelación o un error.

`dispose()` es idempotente: liquida el resultado localmente como `aborted` (no hay cancelación de prompt a nivel de cableado) y cierra después el runtime —una solicitud `shutdown` de protocolo acotada seguida de la escalera compartida stdin-EOF → SIGTERM → SIGKILL hasta la salida real—.

## Mapeo de la razón de parada

El cliente SDK devuelve una actividad de hijo propia en lugar de un resultado de prompt. El provider lee el último `turn/end` durable dentro de esa actividad y lo mapea al vocabulario del seam: `completed` → `completed`, `max-tokens` → `max-tokens`, `aborted` → `aborted`; todo lo demás —`error`, `interrupted`, `disposed`, una variante futura o una actividad sin turno— se mapea a `error`, de modo que una parada no limpia nunca se informa como éxito. Los fallos a nivel de transporte posteriores a la publicación se aplanan a `stopReason: 'error'` a través del sumidero de diagnóstico `onError` (conectado a `ctx.logger.warn`); el contrato del seam prohíbe que `result` rechace.

## Capacidades y contexto

El provider no anuncia funcionalidades de tiempo de arranque (`outputSchema`/`depthLimit`/`toolFilter`/`persona` todas en falso) e informa `inheritsParentContext: false`: el hijo es un runtime nuevo en otro proceso, y la única entrada derivada del parent es el cwd de workspace. Los despliegues de `dsh-tool-subagent` sobre este provider fijan `maxDepth: 'provider-managed'` —el harness hijo es dueño de su propio presupuesto de recursión—.

## Configuración

| Clave | Por defecto | Significado |
|---|---|---|
| `providerName` | `dsh-sdk` | Nombre de registro en `ctx.subagents`. |
| `command` | obligatorio | Ejecutable del que se hace spawn en cada ejecución (el bin del runtime hijo o el exe empaquetado). |
| `args` | `[]` | Argumentos del comando (normalmente la ruta al `cordis.yml` del hijo). |
| `cwd` | cwd de la sesión del parent | Anulación del directorio de trabajo; misma validación que [`subagent-acp`](../subagent-acp/README.es.md). |
| `provider` | `deepseek-official` | Ruta de provider enviada en el `initialize` del hijo. |
| `model` | `deepseek-v4-flash` | Modelo enviado en el `initialize` del hijo. |
| `maxTokens` | por defecto de la ruta adaptador/provider | Tope de tokens de salida por solicitud enviado en el `initialize` del hijo; se aplica al agent raíz del hijo y a sus descendientes en proceso. |
| `env` | `{}` | Entorno explícito del hijo superpuesto sobre un entorno del parent con las credenciales eliminadas (p. ej. el `DEEPSEEK_API_KEY` propio del hijo, o `DSH_CORDIS_CONFIG`). |
| `shutdownTimeoutMs` | `1000` | Límite del intercambio de `shutdown` del protocolo durante la disposición. |
| `disposeEofGraceMs` | `6000` | Período de gracia tras el EOF de stdin antes de la terminación de plataforma. |
| `disposeGraceMs` | `3000` | Período de gracia de confirmación de salida tras la terminación; POSIX espera además este tiempo tras SIGTERM antes de SIGKILL. |

```yaml
- id: subagent-dsh-sdk
  name: '@deepseek-ai/dsh-subagent-dsh-sdk'
  config:
    providerName: dsh-sdk
    command: node
    args: ['./packages/examples/jsonrpc-demo/lib/bin.js', './examples/jsonrpc-agent/cordis.yml']
    maxTokens: 49152
    env:
      DEEPSEEK_API_KEY: !!js process.env.DEEPSEEK_API_KEY
- id: tool-subagent
  name: '@deepseek-ai/dsh-tool-subagent'
  config: { provider: dsh-sdk, toolName: subagent, maxDepth: 'provider-managed' }
```

## Límite de proceso

El entorno del hijo es la base `scrubbedParentEnv()` del seam de [`dsh-subprocess`](../../subprocess/README.es.md) —se descartan los nombres ambientales con forma de credencial y `DSH_*`— con los valores explícitos de `config.env` fusionados después de la depuración. Del hijo se hace spawn a través del cliente SDK y no mediante `ctx.subprocess` (la excepción documentada del README de subprocess para transportes gestionados por SDK), que es por lo que este backend aplica la depuración él mismo. El cableado JSON-RPC es la frontera de serialización real.

El paquete no tiene export por defecto. El unwrapping del loader de Cordis ocultaría si no los metadatos con nombre `inject`; ver [post-mortem 0001](../../../docs/postmortem/0001-acp-default-export-drops-inject.es.md).

## Model Experience

### Solicitud del agent hijo

#### Lo que ve el modelo

El modelo del runtime hijo recibe la tarea autocontenida como su mensaje de usuario, además del system prompt, las herramientas y la sesión nueva configurados por ese propio runtime. No recibe la conversación del parent. Este provider no anuncia funcionalidades opcionales de tiempo de arranque, así que el servicio local rechaza las solicitudes de persona, filtrado de herramientas, aplicación de profundidad o salida estructurada en lugar de omitirlas en silencio.

#### Efecto de tokens

El hijo paga un contexto completo independiente y su propio historial de varios pasos. Estos tokens nunca entran en el contexto del parent.

#### Efecto de KV Cache

Independiente de la caché de solicitudes del parent. Cada hijo SDK solo puede reutilizar prefijos idénticos bajo su propio provider, modelo, composición e historial; los pasos del hijo crecen si no solo mediante añadido.

### Resultado de herramienta del parent, indirectamente

#### Lo que ve el modelo

A través de `dsh-tool-subagent`, el parent recibe solo el texto final del asistente del hijo (o el texto parcial acumulado) o el error exacto de razón de parada de ese consumer, no los mensajes intermedios ni el tráfico de herramientas.

#### Efecto de tokens

La entrada del parent crece solo con el resultado o el error finales, que dependen de los datos y se conservan hasta la compactación. Este provider no añade schema al parent por sí mismo.

#### Efecto de KV Cache

Solo de añadido; el contenido recién visible sigue el prefijo de solicitud reutilizable y no invalida las entradas existentes de la caché KV.

## Limitaciones conocidas y trabajo diferido

- **Un proceso de runtime nuevo por ejecución** — sin pooling; un runtime de harness arranca un árbol de plugins completo, así que el coste de spawn por ejecución es mayor que el del hijo típico del backend ACP.
- **Sin funcionalidades opcionales de tiempo de arranque** — el parent no puede aplicar `outputSchema`, profundidad, filtros de herramientas ni persona dentro del proceso hijo; configura en su lugar el `cordis.yml` propio del hijo.
- **El transcript del hijo permanece en la raíz de sesión propia del hijo** — el log del parent registra solo la llamada/resultado de herramienta de la delegación (la regla de aislamiento del hijo del seam); el canal `session.event` en streaming se consume para extraer la salida y no se puentea al log del parent.
- **Solo procesos hijo locales** — el cwd resuelto es una ruta local; un runtime remoto necesitaría su propio backend.
