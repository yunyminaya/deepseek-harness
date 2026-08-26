# Agent Note: Ciclo de vida del agent y contratos de propiedad

Status: implemented

[English](2026-06-18-agent-lifecycle-and-ownership-contracts.md) | Español

## Problema

Varias limitaciones de ACP y de tool-bash eran síntomas del mismo contrato de propiedad ausente: los plugins podían crear o reanudar agents (agentes) a través de `ctx.agents`, pero no podían ser dueños de un agent de forma independiente ni liberarlo, y las tareas bash de larga duración no llevaban ningún dueño estable en el propio ejecutor. ACP abortaba y esperaba a los agents en la desconexión, pero no podía anular el registro solo del agent de esa sesión; `session/cancel` no podía cancelar trabajo encolado pero aún no iniciado; y `tool-bash` guardaba la propiedad de los trabajos en un `Map` local del plugin, de modo que una recarga HMR podía hacer que una tarea antigua pareciera sin dueño.

## Decisión

Tres cambios de contrato: el cancel consciente de la cola, el disposer `AgentHandle` y el token de dueño de bash.

### 1. `Agent.cancel(cause?)` consciente de la cola

Un verbo nuevo `cancel()` en la interfaz `Agent` — la única primitiva pública de detención. (Originalmente se publicó junto a un `abort()` más acotado, solo de paso; ese verbo se eliminó después por no usarse, dejando `cancel()` como única vía pública de detener el trabajo.) Limpia los FIFOs de encolado y steering de la bandeja de entrada, aborta el turno activo si lo hay, y conserva un marcador previo a la ejecución sin causa, de modo que un prompt cancelado antes de la reclamación nunca se ejecuta mientras un prompt posterior permanece independiente. Una llamada efectiva emite `agent/cancel-requested` con la causa tipada `user | parent` antes de limpiar o abortar; la cancelación en reposo no emite nada y no puede dejar atascado el siguiente prompt. `whenIdle()` alcanza la quietud posterior a la cancelación, y `session/cancel` de ACP se mapea a `user`. La [decisión de cancelación explícita de turno](2026-07-16-explicit-turn-cancellation.es.md) es dueña del contrato actual de causa, duración de la señal y resolución cooperativa.

### 2. El disposer asíncrono `AgentHandle`

`ctx.agents.create`/`resume` (y la interfaz `AgentFactory`) devuelven `AgentHandle = { agent: Agent; dispose(): Promise<void> }`. El disposer es una **capacidad del consumidor** — un observador del registro que sostiene solo el `Agent` a secas no puede desmontarlo. El fiber llamante y el provider de fábrica registrado son co-dueños estructurales: la descarga del llamante impone la propiedad estructurada, mientras que la descarga del provider debe detener las instancias antiguas cuya superficie de dependencias con ámbito se resuelve a través de ese provider. Las tres rutas llegan al mismo teardown memoizado: detener el bucle, esperar su salida y los flushes de reposo (quietud real, no solo el cambio de estado `disposed`), desvincular el agent, desvincular su sesión y desenrollar su scope. Cada ID público vuelve a ser reutilizable cuando se desvincula su entrada exacta del registro; no hay una fase separada de reserva-liberación. Los agents creados por configuración ya son propiedad del fiber `AgentLoop` (el handle se descarta). ACP guarda el disposer de cada sesión nueva en su `SessionRecord` y lo ejecuta en la desconexión o en el teardown del plugin, de modo que una desconexión simple del cliente no deja ningún agent registrado ni ninguna entrada en el almacén de sesiones. Un create que pierde la carrera del cierre libera su handle no publicado.

**El ORDEN del teardown es determinante para la durabilidad**, y la implementación pliega el ciclo de vida de la sesión en el ÚNICO efecto cordis compuesto del agent (`SessionStore.prepare`/`enter`/`announce`, reemplazando una división en efectos hermanos). Una descarga de fiber libera los efectos hermanos de forma concurrente (`Promise.all`), lo que pondría en carrera la eliminación de los hooks de publicación de append del almacén de sesiones contra el `session/flush` de cierre del bucle y descartaría el `turn/end` de cierre; dentro de un solo efecto los disposers se ejecutan como una cadena LIFO ordenada (bucle detenido + `await agent.done` ANTES de que la sesión se desvincule), de modo que el flush final del bucle queda capturado tanto en `dispose()` del handle como en una descarga de fiber. Las notificaciones contenidas `agent/disposed` y `session/disposed` no pueden rechazar la cadena ni saltarse el teardown posterior.

### 3. Token de dueño de bash en la Service Definition

La propiedad de los trabajos en segundo plano se movió de un `Map<string, Agent>` local del plugin `tool-bash` al ejecutor. `ShellExecRequest` gana un `owner?: string` opcional; el `ShellExecSpec` resuelto lo lleva como `owner: string | undefined` obligatorio pero anulable (un dueño olvidado es un `undefined` visible, nunca una propiedad ausente en silencio). El ejecutor guarda el token en su tarea y lo expone mediante un método nuevo `ShellExecutor.ownerOf(id): string | undefined` (NO en el `BashTask` público — una sola ruta de lectura, sin API redundante). `tool-bash` elimina su `Map` por completo: estampa `exec.agent?.id` (el id compartido de registro/sesión) como dueño en `start`, y `bash_output`/`bash_kill` comparan `ctx.shell.ownerOf(id)` con el token del llamante con semántica `!== undefined` (un token de cadena vacía sigue siendo un dueño real). El aviso de finalización encuentra al agent vivo escaneando `ctx.get('agents')?.list()` en busca de `agent.id === ownerToken` (leído vía `ctx.get` — `onJobDone` corre en el fiber de bash, un fiber ajeno, donde el proxy `ctx.agents` lanzaría). Como la propiedad vive ahora en la tarea del ejecutor (liberada con el fiber `dsh-shell`), SOBREVIVE a una recarga HMR de `tool-bash` — cerrando la antigua brecha `XXX(tool-bash-owner-hmr)`. (El listener `onJobDone` sigue teniendo ámbito de efecto sobre el `apply` de `tool-bash`, así que una finalización que aterriza durante la brecha de recarga sigue perdiendo su único aviso — la pérdida preexistente por brecha de recarga — pero la valla de propiedad en sí es a prueba de HMR.)

## Verificación

Estos invariantes se mantienen y quedan fijados por pruebas:

- La desconexión de ACP o el teardown del plugin no dejan ningún agent registrado ni ninguna entrada en el almacén de sesiones para ninguna sesión propiedad del bridge, incluido un create en carrera con el cierre de la conexión.
- `session/cancel` antes de que arranque un prompt encolado impide que ese prompt se ejecute; un prompt aceptado después sigue siendo un turno encolado independiente.
- Una recarga HMR de `tool-bash` NO hace que un trabajo en segundo plano existente sea legible o matable por otra sesión (la propiedad sobrevive en el ejecutor).
- Las demos existentes no ACP siguen funcionando sin gestionar handles explícitamente; los agents creados por configuración siguen siendo propiedad del fiber del plugin `AgentLoop`.

## Los tokens de dueño de sesión son únicos entre los agents vivos

La comparación de tokens de dueño de bash se apoya en que el `Agent.id`/`SessionId` compartido sea único entre los agents vivos. Las operaciones concurrentes con el mismo id pueden prepararse ambas en privado, pero la publicación entra en la sesión y en el agent en orden; `SessionStore.enter()` rechaza un id de sesión viva duplicado, y toda transacción perdedora revierte su estado privado. Un llamante programático no puede, por tanto, publicar dos agents vivos con un único token de sesión. La *política* de acceso (comparación de tokens) permanece en `tool-bash` (el Consumer); la capacidad de bash mantiene `owner` opaco y nunca lo interpreta — la separación correcta de Service Definition / Service Provider / Consumer.

## Alternativas consideradas

- **Un campo público `BashTask.owner`** en lugar del método `ShellExecutor.ownerOf(id)` de la Service Definition — rechazado: una sola ruta de lectura, sin API redundante.
- **Efectos cordis hermanos para el ciclo de vida de sesión del agent** — rechazado: una descarga de fiber libera los efectos hermanos de forma concurrente (`Promise.all`), poniendo en carrera la eliminación de los hooks de publicación de append del almacén contra el `session/flush` de cierre del bucle; la cadena LIFO ordenada del único efecto compuesto es lo que captura el `turn/end` de cierre en ambas rutas de liberación.
- **Un `abort()` separado, solo de paso, junto a `cancel()`** — publicado originalmente y eliminado después por no usarse; `cancel()` es la única primitiva pública de detención ([el Agent Note de la API pública de detención](../simplification/2026-06-20-public-agent-stop-api.es.md)).

## Consecuencias

Esto tocó interfaces públicas (`Agent`, `AgentFactory`, el seam de bash) deliberadamente, no como un parche local de ACP. La entrega síncrona del agent sigue siendo simple; la ruta asíncrona del ciclo de vida es aditiva para los dueños que la necesitan.
