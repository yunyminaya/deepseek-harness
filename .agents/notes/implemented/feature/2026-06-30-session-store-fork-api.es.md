# Agent Note: API de fork de SessionStore

Status: implemented

[English](2026-06-30-session-store-fork-api.md) | Español

## Problema

El log de sesión basado en eventos ya tiene la primitiva que un fork necesita: crear una sesión nueva con un prefijo de eventos semilla y luego derivar el historial del modelo desde ese log sembrado exactamente como hace el replay. Esa primitiva es deliberadamente de bajo nivel: `ctx.sessions.create(id, { seed, meta })` acepta cualquier semilla válida, pero la ramificación ordinaria de sesiones vivas necesita política en torno a qué prefijo se puede copiar, qué metadatos se estampan en el hijo y cómo se clasifican los errores.

El riesgo semántico es la frontera del fork. Una semilla de fork válida y visible para el usuario debe ser contigua y terminar fuera de un turno activo. Bifurcar durante la ejecución copiaría un `turn/start` abierto, posiblemente un `step/start` abierto y posiblemente llamadas de herramienta colgantes. Eso viola las invariantes de ejecución y de transcript del provider, y crea un historial hijo engañoso que parece haber participado en un turno padre sin terminar. El contexto independiente y los eventos solo-log propiedad del plugin son historial bifurcable estable tras un turno cerrado. El [seam de subagente](2026-06-21-subagent-capability-seam.es.md) existente resuelve deliberadamente un problema distinto: los forks de subagente disparados por herramienta suelen ocurrir mientras el turno del padre está abierto, así que `dsh-subagent-fork-in-process` recorta la semilla al último prefijo de turno completado del padre. Un fork de sesión general no debe recortar en silencio; debe o bifurcar la frontera pedida o rechazarla.

## Decisión

`dsh-session` posee la bifurcación ordinaria de sesiones vivas directamente en `ctx.sessions`. No hay un paquete `dsh-session-fork` aparte ni un servicio `ctx.sessionFork`: la API no tiene backend independiente, vocabulario de eventos, ciclo de vida ni comportamiento de persistencia propios, y todo el trabajo durable delega en el almacén de sesiones y los backends de persistencia existentes.

El almacén expone una operación:

```ts ignore-check
type SessionForkSource = Session | SessionId

class SessionStore extends Service {
  fork(source: SessionForkSource, boundary?: number, childSessionId?: SessionId): Session
}
```

`boundary` es el `seq` inclusivo del evento fuente a copiar hasta él. Cuando se omite, toma por defecto el último evento vigente de la sesión fuente; en una fuente vacía, un `boundary` omitido crea un hijo vacío. La validación específica de fork comprueba que la frontera pedida exista y que la última frontera de turno del prefijo seleccionado no sea un `turn/start` sin pareja. El prefijo seleccionado puede por tanto terminar en `turn/end` o en un evento independiente posterior, y luego se clona en profundidad en la semilla del hijo. El hijo hereda el `cwd` de la sesión fuente, estampa `parentSession` con el id fuente y fija `seedLength` a la longitud del prefijo copiado. Cuando se omite `childSessionId`, `SessionStore` genera uno con su política de ids existente.

Un prefijo vacío es bifurcable; toda frontera no vacía debe ser una secuencia existente y segura fuera de un turno abierto. Los errores tipados distinguen fuentes ausentes, objetos obsoletos, ids hijos duplicados, fronteras inválidas y prefijos que terminan durante la ejecución. La validación más amplia del log y la reparación de caídas permanecen con sus propietarios actuales.

### Adaptación de Host y navegador

El RPC `session.fork` del Host acepta `atSeq` como ancla dentro del turno deseado y no como frontera segura inclusiva del almacén. Selecciona el primer `turn/end` en o después de esa ancla; un ancla omitida o pasada-de-final selecciona el último turno completado. Un ancla ya presente en el log pero sin un `turn/end` que la cierre retorna `fork-unavailable` y jamás recurre a un turno anterior, de modo que una acción de mensaje no puede omitir en silencio el mensaje pulsado.

El Host crea el hijo a través del registro de agents con la semilla y el linaje seleccionados, y el alistamiento previo a publicación instala el último provider, modelo y objetivo de razonamiento registrados antes de que el hijo pueda correr. Luego adjunta el hijo al Workspace fuente. Un fallo de adjunción retorna `workspace-attach-failed` con el id hijo ya publicado; el cliente reconcilia ese hijo en su lista de resúmenes antes de mostrar el error. La acción sobre la fila de sesión usa el último turno completado, mientras que una acción de mensaje aporta su seq de evento; ambas abren el hijo tras el éxito, y la expansión de linaje lo hace visible bajo la fuente.

## Alternativas consideradas

**Servicio `ctx.sessionFork` separado.** Una iteración anterior entregó esto como un servicio aparte; sobreajustaba el patrón de seam de capacidad. El código no tenía backend intercambiable, ninguna superficie de eventos adicional, ningún ciclo de vida de propiedad independiente y ningún comportamiento durable más allá de `ctx.sessions.create({ seed, meta })`. Mantener un paquete separado haría que los llamadores descubrieran e instalaran un segundo servicio solo para ejecutar política alrededor de una primitiva del almacén de sesiones.

**Dos funciones: `snapshot()` más `fork()`.** Esto preservaba un cálculo reutilizable de semilla/metadatos, pero el único consumidor soportado creaba una sesión de inmediato. También hacía que la API pareciera más abstracta que la operación concreta que los usuarios necesitan. Un único `fork()` con una `boundary` explícita mantiene la API directa y sigue soportando forks de puntos anteriores.

**Recortar en silencio los turnos abiertos a la última frontera completada.** Eso es correcto para `dsh-subagent-fork-in-process`, donde la delegación suele empezar con el turno del padre abierto y el hijo debe heredar solo el prefijo completado. Es incorrecto para la ramificación ordinaria usuario/sesión porque oculta que el punto de fork pedido no era en realidad una frontera válida y descarta en silencio la cola del turno del padre.

## Consecuencias

La API pública se mantiene pequeña y descubrible: la ramificación de sesión viva es parte de `ctx.sessions`, junto a `create({ seed })`, en lugar de un servicio independiente o una pareja de helpers de dos pasos. La persistencia sigue funcionando a través del comportamiento existente `session/created` y `session/flush`: un hijo de fork empieza su vida con eventos sembrados, así que los backends existentes persisten esa semilla una vez y preservan `parentSession` / `seedLength` en la cabecera.

El alcance v1 aún excluye `session/fork` de ACP, el fork de sesiones persistidas sin cargar, las herramientas orientadas al modelo y los refactors de subagente. Si en el futuro se añade un método ACP, debería anunciar la capacidad solo tras tener cobertura de protocolo e instantáneas; esta Agent Note no añade comportamiento de conexión ACP, así que no se requiere ninguna instantánea ACP. El replay del hijo de fork sigue cubierto por la [Agent Note de pruebas del límite de semilla de fork](../testing/2026-06-22-fork-child-replay-seed-boundary.es.md); las pruebas enfocadas de almacén, Host, carrier y cliente fijan las fronteras y los contratos de reconciliación, mientras que el escenario real de Chromium fija la acción de mensaje ensamblada y el árbol de linaje.
