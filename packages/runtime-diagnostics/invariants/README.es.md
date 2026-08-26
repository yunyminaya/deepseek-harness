# dsh-invariants

[English](README.md) | Español

Servicio de registro configurable para las comprobaciones de invariantes de runtime que poseen los paquetes. El plugin raíz registra `ctx.invariants`; no contiene comprobaciones de producto ni imports de paquetes de producto. Cada paquete del workspace publica un companion `./invariant` que registra su nombre exacto de paquete npm.

## Servicio: `InvariantRegistry` (`ctx.invariants`)

```ts
interface Config {
  enabled?: boolean
  package_allowlist?: string[]
  package_blocklist?: string[]
}
```

Los valores por defecto son `enabled: true`, `package_allowlist: []` y `package_blocklist: []`. Un paquete solo se selecciona cuando el servicio está habilitado, la allowlist está vacía o al menos un patrón de la allowlist coincide con su nombre npm completo, y ningún patrón de la blocklist coincide. Por tanto, las coincidencias de la blocklist prevalecen sobre las de la allowlist.

Cada entrada es una fuente de expresión regular de JavaScript sensible a mayúsculas compilada con `new RegExp(pattern)`. La coincidencia no está anclada salvo que la fuente aporte `^` y `$`; la sintaxis `/pattern/flags` no se analiza. Las entradas en blanco, con espacios alrededor, inválidas o duplicadas dentro de una misma lista hacen fallar el arranque del servicio. Un patrón válido puede no coincidir con ningún paquete cargado actualmente, de modo que la carga posterior y el HMR sigan siendo deterministas.

`ctx.invariants.register(packageName, installer)` reserva un registro activo para el nombre npm completo del paquete, incluso cuando los filtros mantienen inactivo su installer, y devuelve su disposer. Una contribución habilitada se ejecuta en una fibra Cordis hija dedicada. El installer puede declarar sus servicios requeridos a través de `installer.inject` y recibe `fail(message)`, que lanza un `InvariantError` vinculado al paquete que registra. La finalización síncrona o asíncrona del installer se espera antes de que el registro tenga éxito; un fallo dispone la fibra hija y libera la propiedad de forma atómica.

El servicio es dueño de cada fibra de registro, mientras que el disposer devuelto también pertenece a la fibra del companion. Descargar cualquiera de los dos lados elimina los listeners, el estado de trace y la reserva. Un companion puede, por tanto, recargarse y registrar el mismo nombre de paquete sin conservar su estado anterior. Los companions respaldados por sesión reconstruyen su baseline a partir de eventos durables; los companions solo en vivo observan las operaciones que comienzan después de la recarga.

`InvariantError` extiende `Error`, lleva el `code: 'INVARIANT'` estable y expone el `packageName` propietario sin añadir una dependencia de producto al servicio.

La propia Session es dueña del almacenamiento de logs inmutable y válido en superficie en toda composición: toma una instantánea JSON sin pérdidas de cada candidato, valida la cobertura completa de los eventos fuente citados y el reemplazo posicional, restringe el reemplazo de `tool/result` al `content` de un único resultado actual, congela en profundidad el registro aceptado y expone el log mediante instantáneas de array inmutables. El companion de invariantes de `dsh-session` comprueba las reglas restantes entre registros que Session no posee.

## Companions de paquetes

La publicación y el registro son exhaustivos; las aserciones de runtime son deliberadamente no sintéticas. Un companion instala una comprobación solo cuando su paquete posee una relación de eventos observable o una relación de datos mutables relevante. Confirmar un método requerido, un nombre de plugin, una inyección, un efecto o un resultado fijo de función pura es una cuestión de tipos, de carga o de tests unitarios, no un invariante de runtime.

Cuando no existe ninguna relación de runtime plausible, el companion usa un installer vacío con un comentario inicial `No runtime invariant:` específico del paquete que explica el motivo. Es habitual en utilidades puras, implementaciones delgadas cuyo comportamiento ya se observa a través del paquete de su interfaz, paquetes solo de composición, binarios, adaptadores de persistencia cuyos contratos requieren pruebas de crash y round-trip, y paquetes de soporte de tests. La explicación debe revisitarse cuando el propietario gane estado mutable o un protocolo de eventos.

Los companions ejecutables actuales protegen estas relaciones:

| Companion | Comprueba |
|---|---|
| `dsh-session`, `dsh-agent`, `dsh-scope`, `dsh-agent-loop` | Cerramiento de sesión y trace de llamada/resultado, transiciones de estado de agent, conservación FIFO del inbox, subjects con scope y reconstrucción de peticiones de modelo. |
| `dsh-llm`, `dsh-llm-retry`, `dsh-tools`, `dsh-system-prompt` | Gramática del stream, posición y límites durables del reintento, etapas del pipeline de herramientas y resultados congelados, y datos autoritativos del ensamblaje de prompts. |
| `dsh-compaction`, `dsh-hook-protocol`, `dsh-sandbox-policy` | Compactación durable y emparejamiento de hooks, metadatos de compactación y vocabulario de modos de sandbox. |
| `dsh-fs`, `dsh-subagent`, `dsh-workflow` | Identidad de eventos del sistema de archivos, emparejamiento provider/hijo e identidad del ciclo de vida de workflow/agent. |
| `dsh-goal`, `dsh-goal-round-driver` | Acuerdo durable entre fuente y contenido del goal, transiciones de revisión y ciclo de vida, marcas de tiempo, rondas admitidas secuenciales y prompts de continuación reconstruidos. |
| `dsh-permission-presets`, `dsh-user-approval` | Referencias de presets activos y emparejamiento de auditoría de aprobaciones preguntadas/decididas. |
| `dsh-jobs`, `dsh-tool-todo` | Campos de ciclo de vida/propiedad de las instantáneas de tarea y estructura durable del todo completo de la lista. |
| `dsh-time-context` | Las lecturas durables del reloj concuerdan con el turno abierto de la sesión, la siguiente posición de pre-paso y el baseline transcurrido; el tiempo renderizado se analiza y no postdata su evento. |

El entrypoint raíz de cada propietario permanece independiente de los diagnósticos. Cargar el servicio solo no instala comprobaciones de producto, y cargar un companion sin el servicio espera a su inyección declarada de `invariants`.

`pnpm run verify-package-invariants` descubre todos los paquetes del workspace. Rechaza marcadores generados, installers vacíos sin explicar, installers no vacíos que omiten o ignoran el reporter, nombres de registro incorrectos y cableado incompleto de export, publicación, dependencias, referencias TypeScript o bundle. Esta regla de fuente es una comprobación mínima de propiedad; los tests focalizados demuestran la semántica de cada companion ejecutable.

## Composición

```ts
import type { Context } from '@deepseek-ai/cordis'
import InvariantRegistry from '@deepseek-ai/dsh-invariants'
import * as SessionInvariant from '@deepseek-ai/dsh-session/invariant'

declare const ctx: Context

ctx.plugin(InvariantRegistry, {
  enabled: true,
  package_allowlist: ['^@deepseek-ai/dsh-'],
  package_blocklist: ['^@deepseek-ai/dsh-agent-loop$'],
})
ctx.plugin(SessionInvariant)
```

La composición estándar de agent monta el servicio y sus cuatro companions con estado centrales. Las composiciones personalizadas añaden explícitamente companions para otros paquetes cargados cuyos contratos quieran comprobarse; los filtros pueden deshabilitar o seleccionar registros sin cambiar los entrypoints de los paquetes.

Toda topología Vitest ordinaria monta un servicio explícitamente habilitado y el companion del paquete bajo test. Las suites focalizadas cubren observaciones válidas e inválidas de los companions ejecutables, mientras que una topología exhaustiva monta todos los companions para demostrar el cableado de registro y disposición.

## Model Experience

Ninguna, ya que el servicio y los companions observan eventos de runtime e instantáneas mutables sin alterar prompts, mensajes, schemas, streams ni resultados de herramientas.

#### Efecto de KV Cache

Ninguno; las comprobaciones de invariantes no ensamblan ni envían peticiones a providers.

## Limitaciones conocidas y trabajo aplazado

- La reconstrucción de peticiones cubre las peticiones marcadas explícitamente por el loop antes de congelarse; las llamadas directas one-shot al LLM quedan fuera de ese contrato de marcador incluso cuando los llamantes las congelan o les adjuntan un id de sesión.
- Los companions de ciclo de vida solo en vivo no pueden reconstruir operaciones que comenzaron antes de su propia recarga. Las composiciones estándar y de test los montan antes de que comiencen las operaciones correspondientes.
- Los filtros de expresión regular son fijos durante toda la vida del servicio; cambiarlos requiere una recarga ordinaria del plugin Cordis.
