# @deepseek-ai/dsh-subagent-acp

[English](README.md) | Español

El provider ACP ejecuta cada subagente (subagent) en un subproceso nuevo y lo conduce como un cliente del Agent Client Protocol. Es la alternativa fuera de proceso a spawn y fork: el hijo tiene su propio runtime, sesión, configuración de modelo y herramientas.

## Inicio y propiedad

`start(request)` resuelve el directorio de trabajo del hijo y luego ejecuta `spawn` → `initialize` de ACP → `newSession` antes de cumplir. Que cumpla significa, por tanto, que una sesión remota está lista y que la propiedad ha pasado al llamador. Un fallo de spawn, de inicialización, de nueva sesión o de cancelación anterior a la publicación rechaza solo después de que el subproceso haya sido reapado; un fallo de resolución del directorio de trabajo rechaza antes de que se lance nada.

El directorio de trabajo es la sobrescritura `cwd` configurada cuando está fijada; si no, el cwd de la sesión padre que delega — nunca el cwd del propio proceso del servidor, porque un proceso de servidor atiende sesiones de muchos workspaces. El valor derivado del padre debe ser una ruta absoluta que nombre un directorio al que el harness pueda entrar (permiso de búsqueda — lo que necesita un cwd de subproceso), y la misma ruta resuelta se convierte tanto en el cwd del subproceso como en el workspace de `session/new` de ACP.

El run id devuelto se acuña en el espacio de nombres del padre. El id de sesión del servidor hijo permanece privado para las llamadas de wire de ACP porque ACP solo lo garantiza dentro de ese proceso hijo nuevo; usarlo como id de ciclo de vida del padre podría chocar con otra ejecución remota o con un agent local.

Después de la publicación, el provider envía el prompt y recoge el texto de `agent_message_chunk` transmitido en streaming en `SubagentResult.output`. Un fallo de prompt/transporte resuelve con `stopReason: 'error'`, o `aborted` cuando la señal de solicitud requerida o el disposal pidieron la cancelación.

`dispose()` es idempotente. Elimina el listener de la señal, solicita la cancelación de ACP cuando es posible y luego ejecuta la escalera de teardown propia de este backend (`disposeAcpChild`) sobre los verbos del seam: cierra stdin y espera `disposeEofGraceMs` por la quiescencia cooperativa, luego invoca la escalada `terminate()` del handle (SIGTERM, la gracia del spawn, SIGKILL — Windows fuerza la terminación directamente) y espera la prueba de salida de todo el árbol del propietario del subproceso. Cada ejecución usa un proceso nuevo; el pooling de procesos no está implementado.

## Capacidades y contexto

ACP no anuncia capacidades en el momento del inicio porque este proceso no puede imponer la profundidad, el filtro de herramientas, la persona ni el runtime de salida estructurada del hijo remoto. También informa `inheritsParentContext: false`: la sesión remota arranca en blanco, y la única entrada derivada del padre es el cwd del workspace descrito arriba — ningún contexto de conversación cruza la frontera del proceso.

## Configuración

| Clave | Valor por defecto | Significado |
|---|---|---|
| `providerName` | `acp` | Nombre de registro en `ctx.subagents`. |
| `command` | obligatorio | Ejecutable lanzado en cada ejecución. |
| `args` | `[]` | Argumentos del comando. |
| `cwd` | cwd de la sesión padre | Sobrescritura del directorio de trabajo para el proceso hijo y su sesión ACP; debe ser no vacío, un valor relativo se resuelve contra el directorio de lanzamiento del harness en la carga, y el resultado debe nombrar un directorio al que el harness pueda entrar. |
| `permission` | `reject` | Auto-responde a las solicitudes de permiso rechazando o eligiendo la primera opción `allow_once` o `allow_always`. |
| `env` | `{}` | Entorno del hijo explícito superpuesto sobre un entorno padre depurado de credenciales. |
| `disposeEofGraceMs` | `6000` | Gracia positiva tras el EOF de stdin antes de la terminación por plataforma; no puede exceder [`MAX_TIMER_DELAY_MS`](../../util/timeout/README.md.es.md). |
| `disposeGraceMs` | `3000` | Gracia POSIX positiva tras SIGTERM antes de SIGKILL (Windows fuerza la terminación directamente); no puede exceder [`MAX_TIMER_DELAY_MS`](../../util/timeout/README.md.es.md). |

```yaml
- id: subagent-acp
  name: '@deepseek-ai/dsh-subagent-acp'
  config:
    providerName: acp
    command: node
    args: ['--import', 'tsx', './packages/examples/acp-demo/src/bin.ts', '--config', './examples/acp-agent/cordis.yml']
    permission: reject
    env:
      DEEPSEEK_API_KEY: !!js process.env.DEEPSEEK_API_KEY
```

## Mapeo de razones de detención

| ACP | Harness |
|---|---|
| `end_turn` | `completed` |
| `max_tokens` | `max-tokens` |
| `refusal` | `refusal` |
| `cancelled` | `aborted` |
| `max_turn_requests` o desconocido | `error` |

## Frontera del proceso

El hijo se lanza mediante el seam [`dsh-subprocess`](../../subprocess/subprocess/README.md.es.md): el scrub compartido elimina las variables ambientales con forma de credenciales y los nombres ambientales `DSH_*`, y los valores explícitos de `config.env` se fusionan después (un `DEEPSEEK_API_KEY` intencionado sobrevive, y un hecho de despliegue `DSH_*` como `DSH_PERMISSION_MODE` llega al hijo del mismo modo — el scrub descarta solo su homónimo ambiental obsoleto), el stderr se hereda al stream propio del padre, y el disposal aplica la ventana de EOF de este plugin antes de la escalada SIGTERM→SIGKILL propiedad del subproceso y la unión de todo el árbol. El wire de ACP es la frontera de serialización real; los valores de subagentes en el mismo proceso no se clonan defensivamente.

El paquete no tiene exportación por defecto. De lo contrario, el desempaquetado del loader de Cordis ocultaría los metadatos nombrados de `inject`; consulta el [post-mortem 0001](../../../docs/postmortem/0001-acp-default-export-drops-inject.es.md).

## Experiencia del modelo

### Solicitud de agent hijo

#### Qué ve el modelo

El hijo remoto recibe el contenido de la tarea independiente a través de ACP más el prompt de sistema, las herramientas y la sesión nueva de su propio proceso. No recibe ninguna conversación del padre. Este provider no anuncia capacidades opcionales de inicio, así que el servicio local rechaza las solicitudes de persona, filtrado de herramientas, aplicación de profundidad o salida estructurada en lugar de omitirlas en silencio.

#### Efecto de tokens

El hijo paga un contexto completo independiente y su propio historial de varios pasos. Esos tokens nunca entran en el contexto del padre.

#### Efecto de KV Cache

Independiente de la caché de solicitudes del padre. Cada hijo ACP solo puede reutilizar prefijos idénticos bajo su propio provider, modelo, composición e historial; por lo demás, los pasos del hijo crecen solo con añadidos.

### Resultado de herramienta del padre, indirecto

#### Qué ve el modelo

A través de `dsh-tool-subagent`, el padre recibe solo el texto final del asistente transmitido por el hijo o el error exacto de razón de detención de ese consumer, no los mensajes intermedios ni el tráfico de herramientas. Una solicitud ya cancelada antes de la publicación se convierte exactamente en `Error: subagent request was aborted before the ACP child started`; otros fallos de inicio pasan como `Error: <message>`.

#### Efecto de tokens

La entrada del padre crece solo con el resultado final o el error, que depende de los datos y se retiene hasta la compactación. Este provider no añade schema al padre por sí mismo.

#### Efecto de KV Cache

Solo de añadido; el contenido recién visible sigue el prefijo de solicitud reutilizable y no invalida las entradas existentes de la caché KV.

## Limitaciones conocidas y trabajo diferido

- **Un proceso nuevo por ejecución** — el pooling de procesos persistentes es una optimización futura ([la Agent Note del seam](../../../.agents/notes/implemented/feature/2026-06-21-subagent-capability-seam.es.md)).
- **Solo workspaces locales** — el cwd resuelto es una ruta local entregada a un hijo de la misma máquina; el mapeo de workspace para un agent ACP remoto necesitaría su propia capacidad de backend y no está diseñado aquí.
- **Sin capacidades opcionales de inicio** — este provider no puede aplicar el `outputSchema`, el límite de profundidad, el filtro de herramientas ni la persona del harness local dentro del proceso remoto, así que no anuncia ninguna y el servicio rechaza las solicitudes que las requieren.
- **Solo se recoge el texto de `agent_message_chunk` confirmado** — el servidor de automatización mantiene el razonamiento, la actividad de herramientas, los planes y otros datos de traza en el log de la sesión del hijo en lugar de emitirlos por ACP.
- **Los avisos de permiso se auto-responden** (`permission: allow | reject`) — ningún humano ve el `session/request_permission` de un hijo.
