# @deepseek-ai/dsh-subagent-fork-in-process

[English](README.md) | Español

El provider fork crea un hijo en proceso sembrado con los turnos de conversación completados del parent. Comparte toda la mecánica de ejecución con spawn; la semilla de sesión es la única diferencia de comportamiento.

## Límite de la semilla

El turno actual de llamada de herramienta del parent sigue abierto cuando arranca un subagente (subagent): su log contiene la llamada de herramienta del asistente pero no el resultado de herramienta correspondiente ni el `turn/end`. Copiar ese log bruto daría al hijo una sesión inválida y desequilibrada.

Fork calcula por tanto el prefijo contiguo que termina en el último `turn/end`. El hijo ve todos los turnos completados del parent y ninguno del turno en vuelo. Si el parent no ha completado aún ningún turno, la semilla está vacía y el hijo se comporta como un spawn nuevo.

La semilla transfiere solo el historial de conversación. El hijo sigue recibiendo un ámbito de registro plano nuevo; no hereda las restricciones de herramientas ni la autoridad del parent.

## Inicio y capacidades

`start(request)` pasa la semilla de turnos completados a [`startInProcessRun`](../subagent-in-process-driver/README.es.md) y espera la publicación del hijo. El driver compartido es dueño de la cancelación, la profundidad, la personalización, la lectura del resultado y la disposición.

Fork anuncia `{ outputSchema: true, depthLimit: true, toolFilter: true, persona: true }`, idéntico a spawn.

## Config

| Clave | Significado |
|---|---|
| `providerName` | Nombre de registro en `ctx.subagents` (por defecto `fork`). |

Consulta [`dsh-subagent-spawn-in-process`](../subagent-spawn-in-process/README.es.md) para el ciclo de vida de la ejecución, la herencia de modelo y el seguimiento de profundidad —todo compartido—.

## Model Experience

### Historial y envoltura del agent hijo

#### Lo que ve el modelo

El hijo recibe el prefijo de superficie equilibrado de turnos completados del parent y después el contenido nuevo de la tarea tal cual. Una persona configurada sombrea el texto del prompt en el ámbito nuevo del hijo; una restricción de herramientas filtra sus schemas globales de cableado, la búsqueda de ejecutables y los bindings del SDK de Code Mode, pero no las guías independientes. La vista de herramientas y la autoridad del parent no se heredan. Una solicitud opcional de salida estructurada añade su contrato solo-hijo. El turno en vuelo actual del parent queda excluido.

#### Efecto de tokens

El fork duplica el historial completado conservado en solicitudes de hijo separadas; el hijo acumula entonces sus propios tokens de forma independiente. La persona cambia el coste repetido del prompt, el filtrado cambia el coste de schema o de SDK generado, y un fork de primer turno no tiene historial heredado.

#### Efecto de KV Cache

El hijo puede reutilizar el prefijo heredado idéntico byte a byte bajo el mismo provider y modelo. Los cambios de persona, filtro de herramientas, SDK generado o ruta pueden invalidar la reutilización antes que el historial heredado; el historial posterior del hijo es solo de añadido. Las composiciones publicadas vinculan por tanto este provider a `backgroundMode: one-shot`, porque un hijo continuable lleva además la herramienta `report` con ámbito de hijo y su sección de prompt —deltas que preceden al historial heredado y por tanto lo invalidan por completo ([el Agent Note del fork one-shot](../../../.agents/notes/implemented/architecture/2026-08-10-fork-children-stay-one-shot.es.md))—.

### Resultado de herramienta del parent, indirectamente

#### Lo que ve el modelo

El parent recibe solo la salida final propia del hijo a través de `dsh-tool-subagent`, no el prefijo heredado ni el trabajo intermedio.

#### Efecto de tokens

La entrada del parent crece con un resultado final dependiente de los datos conservado hasta la compactación.

#### Efecto de KV Cache

Solo de añadido; el contenido recién visible sigue el prefijo de solicitud reutilizable y no invalida las entradas existentes de la caché KV.

## Limitaciones conocidas y trabajo diferido

- **La semilla es una instantánea única** — el hijo ve los turnos completados del parent en el momento del fork y nada de lo que el parent registre después; no hay intercambio de contexto en vivo.
- **Ninguna composición publicada crea un hijo fork continuable** — `prepareContinuable` sigue implementado y el seam lo acepta, pero todo `cordis.yml` publicado fija `backgroundMode: one-shot` en la herramienta de delegación de fork, así que la ruta continuable del provider no tiene llamador en producción. Reabrirlo exige que el system prompt y los schemas de herramientas del hijo coincidan byte a byte con los del parent, cosa que el [canal de retorno `report`](../tool-subagent-report/README.es.md) impide hoy. Justificación y condición de reintroducción: [el Agent Note del fork one-shot](../../../.agents/notes/implemented/architecture/2026-08-10-fork-children-stay-one-shot.es.md).
