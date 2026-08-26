# @deepseek-ai/dsh-session-projection

[English](README.md) | [中文](README.zh.md) | Español

Service Definition de proyección de sesión y registro de conducción. Es dueño de `ctx.sessionProjections`, el registro que conduce cada unidad de proyección registrada sobre los eventos de sesión confirmados y sirve valores completos terminados a los carriers, actualmente la página de cola del historial del api-proxy y la trama push `session/projection`. Un dominio registra matemática pura; el marco es dueño de la conducción. El [RFC de proyección de sesión](../../../.agents/notes/proposed/architecture/2026-07-27-session-projection-and-command-log.es.md) registra el fundamento de diseño.

## Servicio: `SessionProjectionRegistry` (clave ctx: `sessionProjections`)

### API pública

- `ctx.sessionProjections.register(definition): () => void` Registra la unidad de un dominio. Las claves duplicadas y los `stateVersion` inválidos lanzan; la inscripción es un efecto en la fibra llamante, así que la clave de un plugin de dominio no cargado (con sus celdas en caché) desaparece de las conducciones e instantáneas posteriores — los clientes lo leen como ausencia de capacidad.
- `ctx.sessionProjections.onChanged(listener): () => void` Se suscribe al feed de cambios: una llamada por unidad visible al cliente cuya referencia de estado cambió, por evento confirmado, transportando la vista validada por schema y el seq causante. Ligado a efectos como `register`.
- `ctx.sessionProjections.stateOf(session, key)` Lee el estado de host actual de una unidad registrada sin computar vistas ajenas. El valor devuelto es una referencia en vivo de solo lectura; los llamadores no deben mutarla.
- `ctx.sessionProjections.snapshot(session): ProjectionSnapshot` Un corte síncrono consistente sobre cada unidad registrada visible al cliente — `{ asOfSeq, values }` con `asOfSeq` = el seq del último evento que refleja cada valor (`-1` para un log vacío). El estado solo de host está disponible únicamente a través de `stateOf`.

### Tipos de clave

- `SessionProjectionMap` — la tabla de vistas de cliente fusionable por extensión compartida por los bloques de wire y los hooks de cliente. Los valores son valores completos wire-JSON; el renderizado pertenece al sistema de slots, nunca a esta capa.
- `SessionProjectionStateMap` — la tabla de estado de pliegue de host fusionable por extensión. Toda clave visible al cliente aparece en ambas tablas; las claves solo de host aparecen solo aquí.
- `ProjectionDefinition<K, S>` — `{ key, stateSchema, init(), apply(state, event), wire?, stateVersion }`: una unidad de cómputo síncrona dirigida por estado. `wire` suministra `viewSchema` y `view`; omitirlo hace que la unidad sea solo de host.

## Contrato

- **El marco conduce; el dominio computa.** El registro se suscribe a `session/event` una vez; cada evento confirmado pasa por el `apply` de cada unidad de forma ansiosa. Los dominios no tienen suscripciones. Las celdas (`{state, observedSeq}` por unidad y por sesión, con clave WeakMap) se construyen perezosamente — una unidad registrada después de que fluyeran eventos, o una lectura de una sesión anterior a la inscripción, pliega `init` sobre el log en memoria en el primer contacto.
- **Misma referencia significa cero trabajo.** `apply` DEBE devolver la misma referencia de estado para los eventos que no conciernen a la unidad; la conducción compuerta el feed de cambios con `Object.is`, de modo que los eventos que no coinciden cuestan una llamada y nada aguas abajo.
- **Regla del evento de valor completo (estructural).** Un evento de log que transporta estado DEBE llevar el estado completo posterior al cambio, nunca un delta desnudo — mantiene cada transición trivialmente barata y cada valor servido autodescriptivo (gana el último para los consumidores).
- **Disciplina síncrona de la unidad.** `init`/`apply`/`wire.view` DEBEN ser síncronos; los carriers leen `snapshot()` en el mismo tick que su porción de página, que es lo que hace de `asOfSeq` un corte consistente. Una vista accidentalmente asíncrona devuelve una Promise, que falla `wire.viewSchema.parse`.
- **El estado es plain JSON validado; `stateVersion` es su ancla de invalidación.** La caché de proyección persistida almacena filas `(sessionId, key, ver, seq, val)` y valida `val` con `stateSchema` antes de usarlo; sube `stateVersion` siempre que cambien los campos de estado o la semántica de pliegue. El estado de cada unidad se checkpointea — visible al cliente y solo de host por igual.
- **Ningún vocabulario de wire aquí.** El registro expone solo el feed de cambios y la cara de lectura de instantáneas; los carriers (api-proxy) acuñan sus propias tramas (`session/projection`) y bloques a partir de ellos.
- **Capacidad opcional.** Los plugins de dominio se registran bajo `ctx.inject(['sessionProjections'], …)` para que los ensamblajes headless sin el registro no se vean afectados; los carriers usan `ctx.get('sessionProjections')` y omiten por completo sus bloques/tramas cuando el registro está ausente.

## Rol

Este paquete es dueño de los roles de Service Definition y conducción del seam de capacidad: los plugins de host de dominio (p. ej. `dsh-tool-todo`) contribuyen unidades, los carriers (`dsh-host-apiproxy`) consumen la instantánea y el feed de cambios, y ninguno conoce al otro.

## Experiencia del modelo

Ninguna, ya que el registro solo computa modelos de lectura orientados al cliente del estado de sesión ya registrado y no toca ningún prompt, mensaje, schema, stream ni resultado de herramienta.

#### Efecto de KV Cache

Ninguno; las proyecciones nunca ensamblan ni envían solicitudes de provider.

## Limitaciones conocidas y trabajo diferido

- **Cada página de cola lleva todas las claves visibles al cliente** — aún no existe exclusión por clave ni forma de solicitud de clave perezosa; aceptable mientras los valores sean estados completos a escala de UI (una lista de todo, una instantánea de objetivo), a revisar si el valor de un dominio crece.
- **La tabla de unidades es de todo el proceso, así que la presencia de una clave no es una señal de capacidad por sesión** — una clave registrada por CUALQUIER preset de agent (agente) aparece en la instantánea de cada sesión, incluidas las sesiones cuya propia composición no monta nada que la produzca. Un cliente debe leer el VALOR (`plan.active`, una lista de todo vacía) en lugar de tratar una clave ausente como ausencia de la funcionalidad; una unidad cuyo valor vacío es indistinguible de uno real pertenece en cambio al plano de host, que es por qué `dsh-token-meter` está ahí.
- **La conducción ansiosa toca cada unidad por evento** — barata por construcción (regla de valor completo, compuerta de misma referencia), pero un camino caliente justificaría prefiltros por tipo de evento y por unidad, añadibles sin cambio de contrato.
- **Las celdas del registro viven solo en memoria** — un reinicio reconstruye plegando el log en el primer contacto; las composiciones que montan `dsh-session-projection-cache` siembran ese pliegue desde las filas persistidas en su lugar.
- **La disciplina síncrona de la unidad es solo parcialmente mecánica** — `wire.viewSchema.parse` rechaza una vista que devuelve Promise, pero un `apply` que bloquea o lee estado no-sesión rasgado es una preocupación de revisión; el acompañante de invariantes documenta por qué no existe ninguna comprobación en runtime.
