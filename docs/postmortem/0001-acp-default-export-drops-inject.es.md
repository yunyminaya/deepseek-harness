# Post-mortem 0001: el servidor ACP se bloqueó al conectar — `export default` descartó el `inject` del plugin

[English](0001-acp-default-export-drops-inject.md) | [中文](0001-acp-default-export-drops-inject.zh.md) | Español

Estado: resuelto (corrección en el PR (pull request) #41 `feat/acp-2-bridge`)

## Resumen ejecutivo

Dos errores de integración rompieron ACP a pesar de la cobertura completa de pruebas unitarias: una exportación por defecto hizo que el Loader descartara `inject`, y una búsqueda trazada de un servicio opcional falló a través de un límite de shadow. Las pruebas montadas a mano esquivaban ambas vías. Las correcciones añadieron cobertura sin clave del Loader real y reglas de paquete para las exportaciones de plugins y el acceso a servicios opcionales.

## Resumen

El servidor ACP (`examples/acp-agent`, `@deepseek-ai/dsh-acp`) se bloqueó en el instante en que se conectó un editor real (Zed): la primera petición `session/new` devolvió `Internal error: cannot get property "agents" without inject`, y `session/load` devolvió lo mismo para `sessionPersistence`. El bridge era completamente no funcional en producción a pesar de 178 pruebas unitarias en verde y un 100 % de cobertura de líneas. Dos errores independientes se ocultaban tras la misma cadena de error, y la suite de pruebas pasó por alto ambos por la misma razón: cada prueba montaba el plugin por una vía que no ejercitaba cómo carga realmente ni cómo se resuelven realmente sus servicios.

## Impacto

El servidor ACP no podía crear ni cargar una sola sesión — las dos RPC que un editor invoca primero. Cualquiera que integrara el agente en Zed obtenía un fallo duro inmediato. Sin pérdida de datos (nada se persistió antes del bloqueo); el coste fue enteramente «la funcionalidad no funciona» más el tiempo de depuración para descubrir por qué, dos veces.

## Cronología

- El bridge (RFC 010) se integró con una suite completa de pruebas unitarias para el codec, el transporte en memoria, los mensajes de protocolo generados, las rutas de fallo y el HMR (hot module replacement); una e2e de API real restringida por clave; y una e2e sin clave de pureza de stdout. Todo en verde, 100 % de cobertura.
- Una sesión real de Zed falló de inmediato en `session/new` con `cannot get property "agents" without inject`.
- La investigación persiguió inicialmente una teoría Cordis «traceable/shadow» (plausible, y el mecanismo es real — ver el Error #2), luego instrumentó el recorrido real de fibras (fiber walk) en el `reflect.ts` de vendor y ejecutó el subproceso real. La traza mostró el lanzamiento en `apply()` línea 179 *en el momento de la carga del plugin*, en la fibra ROOT sin shadow — refutando la teoría del shadow para `session/new`.
- Se encontró la causa raíz #1: un `export default apply` accidental. Eliminarlo arregló `session/new`.
- Eliminarlo expuso entonces el Error #2: `session/load` seguía lanzando por `sessionPersistence` — un mecanismo genuinamente distinto (el recorrido de shadow), confirmado al aislar la corrección y volver a ejecutar el subproceso real.

## Causa raíz #1 — `export default apply` descarta el `inject` del plugin (rompió `session/new`)

`packages/acp/acp/src/index.ts` es un *plugin de espacio de nombres*: exporta `name`, `inject`, `Config` y `apply` como exportaciones con nombre separadas, como hace cualquier otro plugin del repositorio (`invariants`, `llm-deepseek`, `tool-bash`, `tui`, …). Pero *además* terminaba con una línea extra que ningún otro plugin tenía:

```ts ignore-check
export const name = 'acp'
export const inject = ['agents', 'sessions', 'sessionPersistence']
export function apply(ctx: Context, config: AcpConfig): void { /* … */ }
// …
export default apply   // ← the bug
```

Cuando se carga un plugin desde `cordis.yml`, el Loader de Cordis normaliza el módulo importado a través de `Loader.unwrapExports` (`vendor/loader/src/index.ts`):

```ts ignore-check
unwrapExports(exports: any) {
  if (isNullable(exports)) return exports
  exports = exports.default ?? exports        // ← prefers `.default`
  if (!exports.__esModule) return exports
  return exports.default ?? exports
}
```

Con una exportación por defecto presente, `exports.default ?? exports` se resuelve a la **función `apply` desnuda**. Una función desnuda no tiene `inject`, ni `name`, ni propiedades `Config` — esas vivían como exportaciones con nombre *hermanas* en el espacio de nombres del módulo, y desenvolver hasta `.default` descartó el espacio de nombres. El Loader construyó entonces la fibra del plugin a partir de un `inject` vacío.

En consecuencia, `apply` se ejecutó en una fibra **sin servicios inyectados**. La primera línea, `const agents = ctx.agents`, recorrió el árbol de fibras (ROOT → Include → Loader → ROOT) y, al no encontrar `agents` en el almacén de ninguna fibra y llegar a la fibra raíz (`runtime === null`), lanzó `cannot get property "agents" without inject`. El bloqueo fue en el *momento de la carga*, no en un manejador de peticiones posterior — la petición solo resultó ser lo que disparaba la carga en la traza del fallo.

**Corrección:** elimina `export default apply`. El Loader usa entonces el espacio de nombres del módulo, respeta `inject`/`name`/`Config`, y `apply` se ejecuta dentro de una fibra que concede de verdad los servicios declarados.

## Causa raíz #2 — la lectura de un servicio opcional dispara la guardia de `inject` a través de un shadow trazable (rompió `session/load`)

Con el #1 corregido, `session/new` funcionaba pero `session/load` seguía lanzando `cannot get property "sessionPersistence" without inject`. Este *sí* es el mecanismo traceable/shadow de Cordis, y merece la pena entenderlo con precisión.

`session/load` llama a `agents.resume(...)`, que delega en `AgentLoop.resume()`, que leía `this.ctx.sessionPersistence`. El `static inject` de `AgentLoop` deliberadamente NO incluye `sessionPersistence` — inyectarlo haría que las demos no persistentes se quedaran colgadas para siempre esperando un backend que nunca carga. El servicio lo proporciona un plugin/fibra hermano separado y se lee de forma oportunista.

El acceso a servicios en Cordis pasa por un proxy de contexto (`vendor/cordis/src/reflect.ts`). Cuando se invoca un método de servicio a través de un *proxy trazable* obtenido de una fibra ajena (aquí: la fibra del bridge llama a `ctx.agents.resume`, y el registro devuelve `this.factory` — el `AgentLoop` — re-envuelto como un proxy trazable nuevo vinculado al llamador), `createShadowMethod` (`vendor/cordis/src/utils.ts`) re-vincula `this` a un objeto *shadow* cuyo `ctx` lleva `[symbols.shadow]` apuntando al contexto de construcción propio de `AgentLoop`. Dentro de `resume`, entonces, `this.ctx.sessionPersistence` se resuelve con el manejador del proxy iniciando su recorrido de fibras desde la fibra del shadow:

```ts ignore-check
// reflect.ts get handler
let fiber = (ctx[symbols.shadow] as Context ?? ctx).fiber   // ← starts at AgentLoop's fiber
while (true) {
  const impl = fiber.store?.[prop]
  if (impl) return getTraceable(ctx, impl.value)
  if (prop in fiber.inject) { /* inactive-context error */ }
  if (!fiber.runtime) throw error                            // ← reached root, throw
  if (fiber.parent[symbols.isolate][prop] !== key) throw error
  fiber = fiber.parent.fiber                                 // ← ancestor-only
}
```

El recorrido es **solo de ancestros**. `sessionPersistence` no está ni en el almacén de la fibra de `AgentLoop` (no está en su `static inject`) ni en ningún ancestro de camino a la raíz (vive en una rama *hermana*), así que el recorrido llega a la fibra raíz y lanza.

¿Por qué no detectaron esto las pruebas en memoria de resume de `AgentLoop`? Porque llaman a `ctx.agents.resume(...)` directamente desde el código de prueba — *fuera de cualquier fibra de plugin*. Allí, `ctx.fiber.runtime` es `null`, así que el manejador del proxy toma un atajo temprano:

```ts ignore-check
if (!ctx.fiber.runtime) return ctx.reflect.get(prop, false)   // ← direct global-store lookup, no fiber walk
```

`ctx.reflect.get(name, false)` es una búsqueda directa en el almacén global de servicios indexado por el símbolo de isolate — ignora por completo la topología de fibras y encuentra el servicio. Así que desde una prueba de nivel superior la lectura funciona; desde dentro de una fibra de plugin real, alcanzada a través de un shadow, lanza. El bridge es exactamente el segundo caso.

**Corrección:** lee el servicio opcional con `ctx.get('sessionPersistence')`, que usa el almacén global indexado por isolate conservando las comprobaciones de estado activo. Las lecturas directas de propiedades siguen siendo adecuadas para los servicios del conjunto de inyección declarado del plugin.

## Por qué todas las pruebas lo pasaron por alto (el fallo real)

Ambos errores comparten una única brecha de proceso: **ninguna prueba ejercitó el plugin a través de su ruta de carga real ni de su topología de llamadas real.**

- El harness en memoria monta el bridge construyendo a mano un objeto de plugin: `ctx.plugin({ name, inject, apply })`. Eso suministra `inject` manualmente, así que nunca puede reproducir el Error #1 — `unwrapExports` solo lo llama el *Loader*, nunca `ctx.plugin`. Ni siquiera `ctx.plugin(NamespaceImport)` lo habría detectado.
- El mismo harness monta todo plano en un único contexto raíz, así que un resume de `AgentLoop` alcanzado desde allí se ejecuta o bien a nivel superior (el atajo `!runtime`) o bien a través de un shadow cuyo origen sigue resolviéndose en la raíz — enmascarando el fallo del recorrido de ancestros del Error #2.
- La única e2e sin clave enviaba `initialize` y comprobaba la pureza de stdout. `initialize` nunca llega a la fábrica, así que pasó de largo frente a ambos errores.
- La única prueba que ejercitaba `session/new`/`session/load` estaba restringida por clave, así que la CI (sin clave) la omitía — y localmente «pasaba» solo porque un `lib/` compilado obsoleto (con el código antiguo) satisfacía por casualidad la resolución de módulos.

El 100 % de cobertura de líneas se cumplió en todo momento. La cobertura demuestra que las líneas *se ejecutaron*; no dice nada sobre si la funcionalidad funciona *tal como se distribuye*.

## Salvaguardas añadidas

- **Se eliminó `export default apply`** (`packages/acp/acp/src/index.ts`) — la corrección del Error #1.
- **`AgentLoop.resume` lee `this.ctx.get('sessionPersistence')`** (`packages/core/agent-loop/src/index.ts`) — la corrección del Error #2, con un comentario que explica la trampa del recorrido de shadow.
- **e2e de `session/new` sin clave sobre stdio real** (`examples/acp-agent/tests/acp.e2e.ts`): arranca el ejemplo como subproceso a través del Loader real y verifica que `session/new` se resuelve. Esto falla ruidosamente con el Error #1 y sin clave de API. Verificado: falla cuando se restaura `export default apply`.
- **`TSX_TSCONFIG_PATH` en el spawn de la e2e**: el subproceso se ejecuta desde un cwd temporal, donde tsx no puede encontrar el mapa `paths` del tsconfig de la raíz del repositorio buscando hacia arriba — así que los imports de dsh-* caían silenciosamente al `lib/` compilado. Apuntar tsx al tsconfig del repositorio hace que la resolución sea independiente del cwd y garantiza que la prueba ejecute *código fuente*, no una compilación posiblemente obsoleta.
- **Regla de [docs/testing.md](../testing.es.md)**: «prueba la ruta de entrada real», la cobertura de líneas no es cobertura de comportamiento — codifica la lección para todo plugin futuro.

## Lecciones

- Un plugin de espacio de nombres y una exportación por defecto son mutuamente excluyentes bajo el Loader de Cordis. Elige la forma de espacio de nombres (`name`/`inject`/`Config`/`apply`) y no añadas `export default` — `unwrapExports` descartará el espacio de nombres.
- Para un servicio que un plugin lee de forma oportunista pero NO declara en `static inject`, usa `ctx.get(name)`, nunca `ctx.<name>`. El proxy de propiedades resuelve con un recorrido de fibras solo de ancestros que falla a través de un shadow ajeno; `ctx.get(name)` es la búsqueda independiente de la topología (y estricta por defecto — un backend inactivo se lee como `undefined` en lugar de devolverse a mitad del teardown).
- Una prueba que construye un plugin a mano no puede validar cómo carga el plugin. Al menos una prueba debe recorrer de extremo a extremo la ruta real de Loader/exportaciones. Cuando la operación principal no llama al modelo, esa prueba no necesita clave de API — así que pertenece a la CI, no detrás de una puerta de clave.
- Confía en la traza, no en la teoría. La elegante explicación del shadow era real pero era el *segundo* error; el *primero* fue un error de exportación de una línea que un `console.error` en el recorrido de fibras encontró en minutos tras horas de razonamiento plausible pero equivocado.
