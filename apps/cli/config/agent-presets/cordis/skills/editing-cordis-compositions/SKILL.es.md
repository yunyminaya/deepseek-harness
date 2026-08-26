---
name: editing-cordis-compositions
description: Úsalo al crear, cambiar o validar una composición Cordis para este harness — escribir o editar un agent preset, añadir o quitar una fila de plugin, decidir si algo pertenece a la composición host o a una sesión, comprobar si un preset creado por ti monta de verdad, o diagnosticar una fila que montó pero no aportó nada.
---

# Editar composiciones Cordis

[English](SKILL.md) | Español

Toda capacidad de este harness es una fila de plugin en un `cordis.yml`. No existe un lenguaje de configuración aparte: cambiar lo que un agent (agente) puede hacer significa cambiar qué filas se componen para él.

## Prohibido

**Nunca edites, borres ni sobrescribas un preset que venga con el despliegue** — el directorio `agent-presets` junto a la configuración del propio despliegue, que suministra `standard`, `code`, `minimal` y `cordis`. Nunca escales el sandbox para alcanzarlo, ni siquiera cuando un cambio allí parezca más rápido. Una actualización sobrescribe esa instalación, y corromper `cordis` deshabilita la propia creación de presets. Leer una composición incluida en el despliegue es la forma prevista de empezar; escribir en una no lo es, y tampoco lo es editar la composición host para sortear una limitación del preset.

Para cambiar lo que hace un preset del despliegue, cópialo y edita la copia. Los presets creados localmente bajo el root de usuario son tuyos: puedes crearlos, editarlos y borrarlos.

## Decide primero el plano

Hay dos planos, y la elección no depende de lo «relacionado con el agent» que algo parezca: depende de si la cosa debe compartirse.

**Composición host.** Los propios registros (`tools`, `systemPrompt`, `agents`, `agent-loop`, `sessions`), todo lo que cruza sesiones (persistencia, consulta de sesión, almacenamiento, ajustes, credenciales, telemetría), la pila de sandbox y aprobación, la ruta del modelo, y el registro de subagentes con sus backends de spawn/fork. Una instancia por proceso.

**Agent preset.** Lo que una sesión aporta a esos registros: sus plugins de herramienta, su persona y sus secciones de prompt, su política de compactación. Una instancia por sesión, montada bajo el ámbito de esa sesión y desmontada con ella.

**Un servicio con un Consumer fuera del plano del agent no puede pasar a un preset.** `subagents` es el ejemplo tipo: el registro responde a consultas entre sesiones para el api-proxy del host, así que una copia por sesión a la vez deja hambrienta a esa fila host — espera para siempre un servicio que nada proporciona — y choca en la segunda sesión, ya que un nombre de provider se registra una sola vez. El preset aporta las *herramientas* de delegación; el registro y sus backends permanecen en el lado host.

Un preset es un directorio que contiene un `agent.cordis.yml`, opcionalmente junto a un `preset.yml` con los metadatos de visualización — `name` y `description` (y, para los presets del despliegue, un `order` del roster). Escribe también los metadatos: un preset sin ellos aparece en todos los selectores con el nombre pelado de su directorio.

Los presets creados localmente viven en un directorio por preset bajo `${DSH_HOME:-$HOME/.dsh}/.agent-presets/`, y el conjunto del despliegue está junto a la configuración del propio despliegue. Usa esos cuando el usuario pregunte dónde mirar. Un despliegue puede configurar otros roots, así que la ruta que lees o editas sale de `list()` o `resolve()` — que es también donde `copy()` informa de lo que acaba de crear.

## El servicio roster

`ctx.agentPresets` es dueño del descubrimiento, la creación y el montaje. Llegas a él montando un plugin temporal que lo inyecta y registra una herramienta para ti — `cordis_mount` devuelve solo el acuse de montaje, así que una herramienta registrada es como te llega la respuesta de un servicio, y se vuelve invocable en tu siguiente paso.

Lee `cordis_inspect what:"api" name:"agentPresets"` para las firmas actuales antes de escribir el código. En lo que se apoya esta skill (destreza):

- `list()` — cada preset con su `id`, su `trust` (`system` para el conjunto del despliegue, `user` para los creados), y la `path` absoluta de su archivo de composición. Así localizas cualquier composición sin conocer la estructura de instalación; el directorio es el padre de esa ruta.
- `read(id)` — el texto de la composición de un preset, sin herramienta de archivos ni ruta.
- `copy(from, id, name?)` — la única escritura de creación (ver más abajo).
- `standingKeyFor(id)` — valida por montaje un preset (ver más abajo).

```js
return {
  name: 'preset-tools',
  inject: ['agentPresets', 'tools'],
  apply(ctx) {
    harness.registerTool(ctx, harness.defineTool({
      name: 'preset_check',
      description: 'Mount-validate one preset by id.',
      parameters: { id: { type: 'string', required: true } },
      output: { schema: { type: 'string' }, render(_a, v) { return [{ type: 'text', text: v }] } },
      async execute(args) {
        try {
          await ctx.agentPresets.standingKeyFor(args.id)
          return 'mounted OK'
        } catch (error) {
          return error.message
        }
      },
    }))
  },
}
```

Desmonta el plugin con `cordis_unmount` cuando termines; es un probe, no una capacidad que dejar montada.

## Crear un preset

1. **Empieza por una copia.** `copy(from, id, name)` copia un directorio de preset completo al root de usuario — composición, metadatos, directorios de skill, recursos. Valida el id contra `[a-z0-9][a-z0-9-]*` (se convierte en el nombre del directorio, así que sin guion inicial), rechaza un id que cualquier root ya suministre, revierte una copia fallida y reescribe el `preset.yml` de la copia para conservar la descripción de la fuente pero descartando su name y su `order` del roster. Prefiérelo a una copia de shell: no requiere escalado del sandbox, deja la copia en el root que este despliegue haya hecho escribible, y la copia es exactamente tan cargable como su fuente. `resolve(id)` nombra después el archivo que creó — esa ruta, no una adivinada, es el objetivo de las ediciones siguientes. `standard` es el coding agent completo y la fuente habitual.
2. **Espera el sandbox de archivos en cada edición posterior a la copia.** El root de presets de usuario está fuera del espacio de trabajo de la sesión, así que bajo la política `workspace-write` por defecto la primera escritura allí se deniega. Solo las escrituras lo necesitan: leer cualquier composición por ruta absoluta no requiere escalado. Reintenta ese comando exacto una vez con escalado de `sandbox_permissions` y una justificación breve — el usuario la ve y la aprueba. Agrupa tus escrituras (un heredoc por archivo) en lugar de escalar muchos comandos pequeños. `copy()` mismo corre en el lado host y no necesita nada de esto; las ediciones sí.
3. **Escribe el `description` de la copia** en `preset.yml`, y su `name` si no pasaste ninguno a `copy()`.
4. **Edita `agent.cordis.yml`** fila por fila, respetando la regla del plano y la regla del realm.
5. **Valida por montaje el resultado** y luego pásalo al usuario para una sesión real — ambas cosas en *Verificar un cambio*.

Una composición escrita desde cero suele olvidar un realm de grupo o una fila de Consumer; una copia empieza siendo cargable.

## La regla que atrapa a la gente

**Una fila que publica un servicio no puede quedar suelta en un preset.** Registrar un servicio sin un realm de aislamiento lo coloca en el realm global al proceso, así que la segunda sesión que monte ese preset choca con la primera. El montaje lo rechaza en lugar de dejar que la colisión salga a la superficie más tarde.

Que una fila publique un servicio no se ve por su nombre, y los README de los paquetes no están en un despliegue instalado. Léeselo al runtime en vivo en su lugar: `cordis_inspect what:"services"` lista cada servicio con la fiber que lo posee, así que un servicio atribuido a una fiber distinta de la fila que estás añadiendo es uno que esa fila consume en lugar de proveer. Para una fila que no está en tu composición actual, valida por montaje y lee el rechazo — nombra el servicio infractor.

Cuando un preset es dueño legítimo de un servicio, envuelve al provider **y a todo Consumer que llegue a él** en un grupo con realm `isolate`. La composición `standard` del despliegue hace esto con `workflows`, que nada fuera de un agent lee — su grupo `delegation`, con las herramientas de delegación omitidas aquí:

```yaml
- id: delegation
  name: cordis:group
  group: true
  isolate:
    workflows: true
  config:
    - id: workflow-worker-thread
      name: '@deepseek-ai/dsh-workflow-worker-thread'
      config:
        provider: spawn
    - id: tool-workflow
      name: '@deepseek-ai/dsh-tool-workflow'
```

`true` significa un realm privado de cada sesión que monta. Una etiqueta de cadena, en cambio, une subárboles en un único realm compartido; `provide()` sigue lanzando excepción en el segundo registro bajo ese símbolo, así que una etiqueta no agrupa instancias y no es lo que un preset necesita.

Un Consumer que quede fuera del grupo resuelve el registro del host, que el preset no pobló, y entonces no aporta nada. La validación de montaje lo detecta como una fila que nunca se activó.

Los realms son para los servicios que un preset posee, no para todo grupo. Una capacidad host que el preset solo consume debe quedar fuera de un realm, o la fila no podrá resolverla: `tool-bash`, `tool-jobs` y `tool-goal` no publican nada y están sueltas en `standard`, que explica en comentarios qué instancia host resuelve cada una y por qué un realm la rompería. Envolver una fila de Consumer en un realm propio es el mismo error que dejarla fuera del realm de su provider.

## Verificar un cambio

**`standingKeyFor(id)` es la comprobación.** Compone de verdad el subárbol de plugins del preset — el mismo montaje que realiza un inicio de sesión, menos el agent — y rechaza las cuatro formas en que una composición falla:

- una fila cuyo paquete no resuelve (`Cannot find package …`);
- una fila cuya config es inválida (`invalid config: $.<field> missing required value`);
- una fila que nunca se activó (`N row(s) did not activate: <id>: waiting for <service>`);
- un servicio publicado en el realm root, que llega como uno de dos mensajes. Un nombre que el host no suministra aterriza en el realm root y la auditoría de montaje lo rechaza: `row(s) published process-global service(s) [<name>]; a preset service must sit behind an isolate realm or move to the host composition` — esta es la forma que adopta un realm olvidado del propio preset. Un nombre que el host ya suministra choca antes de la auditoría: `service "<name>" has been registered at <Owner>`. Ambos nombran el servicio infractor.

Devuelve con normalidad cuando la composición monta. Ejecútalo como comprobación final de una edición terminada, no después de cada línea: un montaje con éxito instala una generación permanente que vive hasta que el proceso sale, mientras que uno fallido libera su subárbol y no deja nada.

**No trates el campo `broken` del roster como validación.** `list()` informa `broken` a partir de una comprobación de forma — el archivo parsea en el dialecto YAML del loader y contiene filas con nombre —, que todas las fallas anteriores superan. Detecta un archivo dañado, no una composición inservible.

`cordis_inspect` informa de la composición de ESTA sesión, así que confirma lo que una fila hace en el runtime en el que ya estás, nunca lo que hará tu nuevo preset.

Tras una validación de montaje limpia, pide al usuario que inicie una sesión con el nuevo preset y confirme la lista de herramientas; el preset decide los schemas de herramienta y las secciones de prompt, y solo una sesión real muestra al agent que produce esa composición.

`cordis_mount` evalúa JavaScript contra el runtime en vivo y desaparece al reiniciar. Es para sondear, no para entregar una capacidad: una capacidad pertenece a un archivo de composición.

## Subagentes de productos nativos

Los providers Codex y Claude Code son Profile Bundles opcionales e independientes. Instala solo los productos que un Profile necesita y reinicia después el Profile para que su Host registre esos providers:

```sh
dsh plugin --profile <name> add @deepseek-ai/dsh-subagent-codex
dsh plugin --profile <name> add @deepseek-ai/dsh-subagent-claude-code
dsh plugin --profile <name> remove @deepseek-ai/dsh-subagent-codex
dsh plugin --profile <name> remove @deepseek-ai/dsh-subagent-claude-code
```

Cada Bundle es dueño de la disponibilidad de su Host; el preset concede aparte a un Agent su herramienta de delegación ordinaria. Nunca muevas un provider de producto al preset y nunca añadas un campo de ajustes específico del producto. Quitar un paquete retira solo ese provider en el siguiente arranque del Profile.

Copia estas plantillas deshabilitadas de un preset completo del despliegue y quita `disabled` solo para los productos que el usuario pidió:

```yaml
- id: tool-subagent-codex
  name: '@deepseek-ai/dsh-tool-subagent'
  disabled: true
  config:
    provider: codex
    toolName: subagent_codex
    backgroundMode: one-shot
    maxDepth: provider-managed

- id: tool-subagent-claude-code
  name: '@deepseek-ai/dsh-tool-subagent'
  disabled: true
  config:
    provider: claude-code
    toolName: subagent_claude_code
    backgroundMode: one-shot
    maxDepth: provider-managed
```

Para instancias adicionales con nombre de Codex o Claude Code, monta una fila de provider separada en el plano host por cada instancia con un `providerName` único, y añade después una fila de herramienta de preset separada cuyo `provider` coincida exactamente con ese nombre y cuyo `toolName` también sea único. Conserva las filas del despliegue para los nombres por defecto `codex` y `claude-code`; no reutilices una fila de herramienta para varios providers ni derives ninguno de los dos nombres de los ajustes de permiso o de entorno.

Las dos filas son independientes. Dejar ambas deshabilitadas conserva el preset copiado, habilitar una expone solo esa herramienta de producto, y habilitar ambas expone las dos. El `dsh` de producción no instala ninguno de los dos providers opcionales: antes de habilitar una fila, instala el Bundle `@deepseek-ai/dsh-subagent-codex` o `@deepseek-ai/dsh-subagent-claude-code` correspondiente en el Profile y reinícialo. Cada Bundle registra su provider por defecto inactivo y usa exclusivamente su CLI de plataforma local fijada en el paquete; las instancias adicionales con nombre usan filas extra del plano host del mismo paquete instalado. Un preset no puede proveer esa dependencia host. `backgroundMode: one-shot` mantiene en primer plano las llamadas omitidas o `false` y permite que un `run_in_background: true` explícito devuelva un Job id genérico. Los presets completos ya llevan `tool-jobs`, mientras que el host base lleva el registro de jobs; conserva ambos para que `job_output`, `job_list`, `job_kill`, la cancelación y los avisos de finalización sigan disponibles. Instalar un Bundle o componer una fila de preset no inicia un producto, no autentica una cuenta, no selecciona un modelo, no sondea credenciales ni gestiona ajustes nativos del producto.

## Qué no mover a un preset

`agent-loop` registra la única factoría de agents y lanza excepción ante una segunda. Los registros son dueños de la capa por sesión y no pueden ser ellos mismos por sesión. La persistencia de sesión debe quedarse en el lado host o la lista de sesiones se fragmenta. Las filas de sandbox, aprobación y permiso son una frontera deliberada: un preset es exactamente tan privilegiado como los plugins que nombra, así que dejar que uno relaje su propio confinamiento anularía el confinamiento.
