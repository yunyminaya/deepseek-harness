# Agent Note: Plugin de cliente MCP — conectar a servidores MCP externos y puentear sus herramientas

Status: implemented

[English](2026-07-07-mcp-client-plugin.md) | Español

## Problema

El harness no tenía forma de consumir herramientas del ecosistema MCP (Model Context Protocol). MCP es el estándar emergente para servidores de herramientas — GitHub, sistemas de archivos, bases de datos, búsqueda de código y cientos de servidores comunitarios exponen herramientas vía MCP. Los usuarios quieren apuntar el harness a uno o más servidores MCP y que sus herramientas aparezcan como herramientas nativas visibles para el modelo, sin escribir código de pegamento por servidor.

`ToolRuntime` ya acepta definiciones de herramientas JSON Schema en bruto (documentado en el README de `dsh-tools`: «Raw JSON-Schema tool definitions (from MCP servers) are still accepted by `ToolRuntime.register()` directly»), y el recetario (cookbook) de extensiones esboza el patrón previsto («MCP | un plugin por servidor: descubrir herramientas → `ctx.tools.register()`»). La infraestructura estaba lista; faltaba el plugin puente.

## Decisión

### Paquete

Un solo paquete `@deepseek-ai/dsh-mcp-client` en `packages/mcp/mcp-client/`. Sin la división en tres paquetes del capability seam — no hay una segunda implementación de cliente MCP previsible, y la convención es «no dividas preventivamente» ([Agent Note de capability seams](../architecture/2026-06-13-capability-seams.es.md)).

### SDK

Usar el [`@modelcontextprotocol/sdk`](https://github.com/modelcontextprotocol/typescript-sdk) oficial (`Client`, `StdioClientTransport`, `StreamableHTTPClientTransport`). El harness no implementa su propio JSON-RPC — coherente con cómo ACP delega en `@agentclientprotocol/sdk`.

### Alcance

Solo MCP Client (sin lado de servidor — ACP ya cubre el rol de «exponer el harness como un agent»). Puentear solo **Tools** — Resources y Prompts quedan diferidos (exigen mecanismos de consumo del lado del harness que aún no existen, y el espacio de diseño es grande).

### Forma del plugin

Plugin de espacio de nombres (exports con nombre `name`/`inject`/`Config`/`apply`, sin `export default`). `inject: ['tools']`. Cada servidor MCP es una instancia de plugin en `cordis.yml` — el mismo paquete cargado N veces con configs distintas, como `dsh-tool-subagent`.

### Configuración

Unión discriminada plana sobre el campo `transport`:

```typescript
interface StdioConfig {
  transport: 'stdio'
  serverName: string          // required namespace, ^[A-Za-z0-9_-]{1,32}$
  command: string
  args?: string[]
  env?: Record<string, string>
  cwd?: string
  toolCallTimeoutMs?: number  // default 60_000
}

interface StreamableHttpConfig {
  transport: 'streamable-http'
  serverName: string          // required namespace, ^[A-Za-z0-9_-]{1,32}$
  url: string
  headers?: Record<string, string>
  toolCallTimeoutMs?: number  // default 60_000
}

type Config = StdioConfig | StreamableHttpConfig
```

`serverName` es la identidad local estable que da espacio de nombres a las herramientas de este servidor en el nombre visible para el modelo (más abajo). Es deliberadamente configuración de usuario, NO el `serverInfo.name` remoto: el nombre remoto es entrada no confiable, no es único entre despliegues (las instancias de prod y staging de un servidor reportan el mismo nombre) y puede cambiar en una actualización del servidor — nada de lo cual puede renombrar silenciosamente herramientas visibles para el modelo. Un `serverName` duplicado entre instancias vivas es un error de configuración: la instancia posterior falla al cargar con un mensaje accionable, nunca con sombreado u omisión silenciosos. Un `serverName` corto (`gh`) es también la perilla para acortar los nombres públicos.

Ejemplo de uso en `cordis.yml`:

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
      Authorization: *** `Bearer ${process.env.MCP_TOKEN}`
```

El modelo ve `mcp__github__create_issue`, `mcp__github__search_code`, `mcp__web__search`.

### Ciclo de vida

Al boot desde `cordis.yml`. El HMR (`@cordisjs/plugin-hmr`) proporciona el intercambio en caliente: editar la entrada yml dispara la disposición de la instancia antigua (desconecta, desregistra herramientas) y la creación de una nueva (conecta, descubre, registra). Sin API dinámica de runtime por ahora. Los nombres públicos son funciones puras de `(serverName, rawName)`, así que un intercambio HMR que conserve `serverName` recrea nombres visibles para el modelo idénticos — el historial de sesión y las reglas de permiso siguen siendo válidos — y añadir o quitar un servidor no relacionado jamás renombra una herramienta existente.

### Descubrimiento y registro de herramientas

Cada herramienta MCP tiene dos nombres:

- `rawName` — el `Tool.name` MCP exacto, usado solo en el cable (`tools/call`).
- `publicName` — el nombre visible para el modelo, globalmente único, registrado en el `ToolRuntime`:

      mcp__<serverName>__<rawName>

Esta forma calificada por servidor es el estándar de facto entre los clientes de agentes multi-servidor — todo producto de usuario final encuestado califica las herramientas MCP por servidor ([Claude Code](https://code.claude.com/docs/en/agent-sdk/mcp#tool-naming-convention) `mcp__github__list_issues`, [Codex](https://openai.com/index/unrolling-the-codex-agent-loop/) `mcp__weather__get-forecast`, [Gemini CLI](https://geminicli.com/docs/tools/mcp-server/#3-tool-naming-and-namespaces), [VS Code](https://github.com/microsoft/vscode/blob/ab9ec62c6a61e429a9abd612ff220c3f4834c9ea/src/vs/workbench/contrib/mcp/common/mcpServer.ts#L217-L260), [Cline](https://github.com/cline/cline/blob/52fdbb1d72f7324a28142a7ba7678d4b53c902f4/sdk/packages/core/src/extensions/mcp/name-transform.ts#L20-L35), [Roo Code](https://github.com/RooCodeInc/Roo-Code/blob/b867ec9145750d0ae1ff7f02d35406e9bf2a0b16/src/utils/mcp-name.ts#L117-L140), [Goose](https://github.com/block/goose/blob/b3a012cbdde854b0fe14f95b1c48543bf6517c0a/crates/goose/src/agents/extension_manager.rs#L1391-L1441), [OpenCode](https://github.com/anomalyco/opencode/blob/d199b1bff90282a4f9cd6251b5fc7b16875a52f6/packages/opencode/src/mcp/catalog.ts#L117-L120)); la escritura exacta `mcp__<server>__<tool>` sigue a Claude Code y Codex. El marcador `mcp__` mantiene los registros MCP fuera del espacio de nombres de las herramientas nativas y da a las reglas de permiso/telemetría una forma estable (`mcp__*`, `mcp__github__*`).

1. Al conectar: drenar la paginación de `client.listTools()`, derivar el `publicName` de cada herramienta y registrar cada una como `ToolDefinition` en bruto vía `ctx.tools.register()`. El JSON Schema y la descripción MCP pasan sin cambios (sin conversión al DSL de `defineTool`); solo se reemplaza el `name` visible para el modelo.
2. Escuchar `notifications/tools/list_changed` → re-ejecutar la misma sincronización (disponer la generación anterior, registrar la nueva). Los nombres deterministas significan que las herramientas sin cambios conservan sus nombres a través de las re-sincronizaciones.
3. El ejecutor captura `rawName`; el nombre público jamás se envía al servidor ni se analiza para recuperar el nombre en bruto.
4. Sin `presentCall`/`presentResult` — los consumidores de UI usan el respaldo genérico de tarjeta neutral al provider.
5. Las herramientas son transparentes en el system prompt — sin anotación «[via MCP]» más allá del nombre mismo.

### Normalización del nombre público

MCP permite nombres de herramienta de hasta 128 caracteres, incluido `.`; el contrato de nombres de función de DeepSeek permite `[A-Za-z0-9_-]` y como máximo 64. Los nombres públicos se normalizan de forma determinista: los caracteres inválidos se vuelven `_`, y cuando el reemplazo o el truncamiento cambiaron el nombre, se añade un hash SHA-256 de 12 caracteres hex de la identidad `(serverName, rawName)` para que identidades MCP distintas jamás puedan colapsar en el mismo nombre público:

```typescript
function publicToolName(serverName: string, rawName: string): string {
  const joined = `mcp__${serverName}__${rawName}`
  const normalized = joined.replace(/[^A-Za-z0-9_-]/g, '_')
  if (normalized === joined && normalized.length <= 64) return normalized
  const hash = sha256(`${serverName}\0${rawName}`).slice(0, 12)
  return `${normalized.slice(0, 64 - 13)}_${hash}`
}
```

### Manejo de conflictos de nombre

MCP garantiza la unicidad de nombres de herramienta solo [dentro de un servidor](https://modelcontextprotocol.io/specification/2025-11-25/server/tools#tool-names); las colisiones entre servidores son la norma, no la excepción (un [estudio de Microsoft Research](https://www.microsoft.com/en-us/research/blog/tool-space-interference-in-the-mcp-era-designing-for-agent-compatibility-at-scale/#namespacing-issues-and-naming-ambiguity) de 1.470 servidores encontró 775 nombres de herramienta colisionantes; `search` solo aparece en 32 servidores, y el servidor oficial de GitHub publica el `create_issue` desnudo). El espacio de nombres siempre activo hace las colisiones estructuralmente imposibles en lugar de manejarlas en el momento de la colisión:

- Dos servidores que publican `search` coexisten como `mcp__github__search` y `mcp__web__search`.
- Una herramienta nativa del harness llamada `search` no se ve afectada.
- Un `serverName` duplicado en la config falla la instancia posterior al cargar (véase Configuración).
- Un servidor que lista el mismo nombre de herramienta dos veces es una lista de herramientas inválida: la sincronización lanza y la generación anterior permanece registrada.
- Un conflicto de registro durante el intercambio solo puede significar que una herramienta ajena ocupa el espacio de nombres `mcp__<serverName>__` de este servidor: la generación parcial se revierte (cero herramientas de este servidor) y el error se registra ruidosamente.

Las herramientas jamás se omiten en silencio; qué herramientas están disponibles nunca depende del orden de carga de plugins.

### Invariantes de nombres

1. Toda herramienta MCP tiene la identidad estable `(serverName, rawName)`; toda identidad activa tiene exactamente un nombre público.
2. Los nombres públicos son deterministas, globalmente únicos y cumplen el contrato DeepSeek de 64 caracteres `[A-Za-z0-9_-]`.
3. El `tools/call` de MCP siempre recibe el nombre en bruto original.
4. Conectar, desconectar o re-sincronizar un servidor no relacionado jamás renombra una herramienta existente.
5. El orden de registro jamás determina qué herramienta está disponible.

### Ejecución de herramientas

Un manejador `execute` unificado para todas las herramientas de un servidor MCP:

1. Resolver `rawName` (el ejecutor lo captura) y llamar a `client.callTool({ name: rawName, arguments }, { signal: exec.signal })` con el timeout configurado — el nombre público jamás se envía al servidor.
2. Preservar el éxito canónico como `{ content: JsonValue[], structuredContent? }`; los bloques MCP JSON completos siguen siendo el valor programático/Code Mode. `isError: true` lanza antes de cualquier persistencia de imagen para que el registro posea la vía de fallo.
3. Preparar una proyección Native ordenada separada. Los tramos de texto se unen con `'\n'`; los enlaces de recurso preservan nombre y URI como texto; los bloques de audio, los recursos embebidos, los bloques malformados y los tipos desconocidos se convierten en diagnósticos explícitos. Si existe alguna imagen, el puente decodifica estrictamente el lote completo, resuelve la ruta exacta más reciente del agent que llama, exige un almacén de adjuntos más la entrada de imagen explícita del modelo, y delega la validación de todos los miembros y la persistencia ordenada en `AttachmentStore.saveImages()`. Cualquier negativa de decodificación, capacidad o almacenamiento renderiza cada imagen como texto de diagnóstico y no devuelve referencias parciales.
4. Mantener `output.render` síncrono y puro. El ejecutor escenifica su proyección más rica en un `WeakMap` local de la generación con clave por la ejecución exacta; `finalizeContent` la instala solo cuando el resultado post-ejecución del registro conserva todavía el valor canónico original y el contenido de respaldo. Un bloqueo de política, un reemplazo de valor o de contenido sigue siendo autoritativo, y una re-sincronización no puede dejar que una generación más vieja consuma estado de ejecución nuevo.
5. Code Mode recibe el valor canónico intacto. Su puente de despacho genérico difiere una secuencia de contenido final con éxito que contenga una imagen a través del resultado externo de `run_code`, de modo que MCP no requiere un caso especial privado de token de padre.
6. Cancelación: `exec.signal` (de la cancelación del agent loop) se propaga al `callTool` del SDK de MCP, a la búsqueda exacta de modelo y a la compuerta previa al almacenamiento.

### Entorno del subproceso (transporte stdio)

Construir el entorno hijo a partir de la base compartida `scrubbedParentEnv()` del seam de subprocesos, que elimina los nombres ambientales que coinciden con `/KEY|PASSWORD|SECRET|TOKEN/i` y los nombres ambientales `DSH_*`, y luego fusionar `config.env` encima. Las anulaciones de entorno explícitas sobreviven al fregado.

### Desconexión / caída

Un supervisor de conexión por instancia reconecta automáticamente tras una conexión perdida con retroceso exponencial acotado y un presupuesto de intentos por incidencia, re-ejecutando el descubrimiento al tener éxito; el agotamiento desregistra las herramientas del servidor y se detiene hasta la recarga. La [Agent Note de auto-reconexión](2026-08-06-mcp-client-auto-reconnect.es.md) posee esa decisión, incluido el bloque de config `reconnect` y la exclusión voluntaria `reconnect.enabled: false` que restaura la recuperación manual por HMR/reinicio.

## Alternativas consideradas

### Lado MCP Server (exponer las herramientas del harness a clientes MCP externos)

Diferido. El puente ACP ya expone el harness como servidor de agent. Añadir una capa de servidor MCP duplicaría eso con un protocolo distinto, y la necesidad principal del usuario es consumir herramientas externas, no exponerlas.

### División en tres paquetes del capability seam (interfaz / impl / consumidor)

Rechazado. No hay una implementación alternativa de cliente MCP previsible — MCP tiene un protocolo, un SDK. La convención es «no dividas preventivamente» hasta que aparezca una segunda implementación.

### Auto-reconexión con retroceso exponencial

Rechazada para v1: añadía un estado de disponibilidad parcial (herramientas registradas pero temporalmente no funcionales), y las caídas de stdio suelen indicar problemas de configuración que el reintento no puede arreglar; HMR era la vía de recuperación. La retroalimentación operativa revirtió el diferimiento — la [Agent Note de auto-reconexión](2026-08-06-mcp-client-auto-reconnect.es.md) la implementa con un presupuesto acotado por incidencia y una exclusión voluntaria.

### Puentear Resources y Prompts

Diferido. Resources necesitan un mecanismo del lado del harness para decidir CUÁNDO inyectar contenido (¿system prompt? ¿bajo demanda? ¿disparado por el modelo?). Prompts necesita un concepto de «plantilla de prompt» del que el harness carece. Ambos exigen su propio diseño; Tools son el punto de partida de alto valor y bajo riesgo.

### Nombres de herramienta en bruto visibles para el modelo con un `toolPrefix` opcional

Rechazado — esta era la propuesta original, basada en la premisa de que «la mayoría de los servidores MCP ya usan prefijos semánticos en sus nombres de herramienta (p. ej. `github_create_issue`)». La premisa es falsa: el servidor oficial de GitHub publica `create_issue`, el servidor de referencia de sistema de archivos `read_file`, Sentry `search_issues` — y el estudio de Microsoft citado arriba muestra que las colisiones son comunes a escala de ecosistema. El prefijado en el momento de la colisión (o advertir-y-omitir) hace además que el conjunto de herramientas disponibles dependa del orden de carga de plugins, y una herramienta podría renombrarse en silencio al añadir un servidor no relacionado — invalidando el historial de sesión y las reglas de permiso a mitad de conversación. Ningún producto multi-servidor encuestado embarca nombres en bruto.

### Espacio de nombres solo de servidor (`github__create_issue`, sin marcador `mcp__`)

Rechazado para v1. Evita las colisiones entre servidores pero no separa los registros MCP de las herramientas nativas del harness, y renuncia a las formas de política MCP amplias (`mcp__*`). El marcador cuesta 5 caracteres; la escritura `mcp__<server>__<tool>` coincide con Claude Code y Codex, maximizando la familiaridad del modelo. Si el ToolRuntime creciera después con espacios de nombres conscientes de la fuente, se podría reconsiderar eliminar el marcador literal como un cambio de política de nombres.

### Derivar el espacio de nombres del `serverInfo.name` anunciado por el servidor

Rechazado. El nombre remoto no es confiable, no es único entre despliegues y puede cambiar en una actualización; la identidad de herramienta y las reglas de permiso no deben seguirlo en silencio. El espacio de nombres es configuración local.

### Preservar múltiples TextBlocks en el resultado de herramienta

Rechazado. `flattenText()` en el serializador de DeepSeek usa `join('')` (sin separador) al aplanar `ContentBlock[]` al formato de cable. Múltiples bloques de texto perderían silenciosamente los límites entre bloques — un bug de corrección. Todas las herramientas existentes devuelven un único TextBlock; el puente MCP hace lo mismo.

### Reemplazar el resultado MCP canónico con `ContentBlock[]` del core

Rechazado. Los llamadores programáticos necesitan bloques MCP completos del protocolo y `structuredContent`, mientras que los consumidores Native necesitan imágenes del core duraderas en lugar de base64. Un valor canónico del protocolo más una proyección separada preserva ambos contratos.

### Añadir un servicio genérico RichContent o hacer I/O en `output.render`

Rechazado. El core ya posee el vocabulario de contenido neutral al rol, y un segundo servicio duplicaría sus contratos de registro y orden. `output.render` es puro, síncrono y reproducible, así que el I/O de adjuntos pertenece a la ejecución asíncrona con una entrega de finalización exacta.

### Dejar que cada herramienta que devuelve imágenes trate especial a los padres Code Mode

Rechazado. Eso acopla las herramientas hoja a los internos de las herramientas compuestas y pierde futuras herramientas ricas. El puente genérico de Code Mode observa el contenido final post-política y reenvía los resultados con imágenes de forma uniforme.

## Pruebas

La cobertura se nombra por nivel; cada comportamiento vive en el nivel más barato que pueda expresarlo.

- **Unitarias** (`tests/mcp-client.spec.ts`, `tests/apply.spec.ts`, SDK de MCP con mock): el algoritmo `publicToolName` (limpio, normalizado, truncar-y-hashear, determinismo, separación de identidades distintas), la disciplina de cable en bruto vs público, la coexistencia entre servidores y con herramientas nativas, el fallo de carga por `serverName` duplicado y la liberación de reserva, el rechazo de listas de herramientas inválidas, el intercambio/reversión de generaciones, la retención ante re-sincronización fallida, los resultados canónicos sin pérdida, el orden rico mixto, los lotes malformados atómicos, la negativa exacta de capacidad/almacén, los diagnósticos explícitos sin imagen, la precedencia de política post-ejecución, la cancelación y la validación del schema de config. La cobertura del 100% por archivo es la compuerta del paquete.
- **E2E** (`tests/mcp-client.e2e.ts`, sin clave): el protocolo MCP real contra el servidor fixture del repo, `@modelcontextprotocol/server-everything` y `@modelcontextprotocol/server-filesystem` sobre stdio, y contra un servidor `StreamableHTTPServerTransport` en proceso sobre Streamable HTTP — descubrimiento bajo el espacio de nombres, normalización de nombres con puntos de extremo a extremo, round-trips de ejecución, guardado/lectura de imágenes duradero con base64 retenida solo en el valor canónico, negativa explícita sin ruta de imagen, rechazo de `serverName` duplicado y disposición.
- **Instantáneas**: el ejemplo ACP ensamblado posee el transcript de imagen en línea visible en el transporte y el transcript de reenvío de imágenes de Code Mode; la E2E del paquete posee el cable MCP real porque la instantánea ejecutable debe seguir sin clave y determinista en lugar de lanzar paquetes de servidor de terceros. Las tarjetas de herramientas MCP siguen usando el respaldo de tarjeta genérica y no requieren instantánea de UI específica del paquete.

## Consecuencias

- Una entrada de `cordis.yml` por servidor MCP es todo el coste de integración: `serverName: filesystem` + un comando stdio (o una URL Streamable HTTP) pone `mcp__filesystem__read_file` en la lista de herramientas del modelo, invocable, con el `read_file` en bruto en el cable.
- Los nombres públicos son parte del historial de sesión y de las APIs de permiso/configuración; el algoritmo de nombres es un contrato de v1 fijado por tests, y cambiarlo tras el lanzamiento es un cambio disruptivo.
- El calificador `mcp__<serverName>__` cuesta tokens en cada nombre. Aceptado: las descripciones y los JSON schemas dominan los tokens de definición de herramienta, y el calificador compra identidad estable, aislamiento de colisiones y formas de política MCP amplias (`mcp__*`, `mcp__github__*`).
- **Estabilidad del SDK de MCP**: el `@modelcontextprotocol/sdk` sigue evolucionando; los cambios disruptivos exigen actualizar el puente. La versión está fijada, y el SDK está ampliamente adoptado (Claude Desktop, Cursor, VS Code), así que los cambios disruptivos difícilmente serán silenciosos.
- **Calidad del schema de herramientas**: los servidores MCP pueden exponer herramientas mal descritas (descripciones vagas, JSON schemas incompletos). El harness las deja pasar tal cual — basura entra, basura sale; eso es responsabilidad del autor del servidor, no del puente.
- **Gestión del proceso stdio**: un servidor MCP que ignora las señales podría atascar la disposición. La disposición de fibers de Cordis tiene quietud acotada; un transporte atascado acaba agotando el timeout a nivel del framework.
- La recuperación de caídas es automática dentro del [presupuesto de reconexión](2026-08-06-mcp-client-auto-reconnect.es.md); la recarga manual sigue siendo la vía tras el agotamiento o con `reconnect.enabled: false`.
- Las cargas de imagen pueden entrar en el contexto del modelo solo a través del almacén de adjuntos duradero compartido y una capacidad de ruta positiva exacta. Las cargas de audio y recursos embebidos permanecen locales a la ejecución con diagnósticos explícitos.
