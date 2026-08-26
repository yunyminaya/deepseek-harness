# Schedule local a la sesión

[English](schedule.md) | Español

Schedule es el dueño de los recordatorios duraderos que vuelven a la Session original en vivo como turnos de conversación ordinarios posteriores. La [Agent Note de Schedule duradero](../../.agents/notes/implemented/feature/2026-08-05-durable-web-schedule.es.md) es la dueña de las decisiones de persistencia y ciclo de vida, la [entrega conversacional](../../.agents/notes/implemented/simplification/2026-08-09-conversational-schedule-delivery.es.md) es la dueña del límite sin recibo, el [límite de zona horaria explícito](../../.agents/notes/implemented/simplification/2026-08-09-explicit-schedule-time-zone.es.md) es el dueño de la interpretación local al navegador, y el [Schedule de tasa fija acotada](../../.agents/notes/implemented/simplification/2026-08-09-bounded-fixed-rate-schedule.es.md) es el dueño de la recurrencia. Esta página registra las formas duraderas y visibles para el modelo de [`packages/schedule/schedule/src/types.ts`](../../packages/schedule/schedule/src/types.ts); el [README del paquete](../../packages/schedule/schedule/README.es.md) es el dueño de la composición, del comportamiento de las herramientas y del encuadre exacto de los recordatorios.

## Registros duraderos

`ScheduleId` es un [id con marca](core.es.md#branded-ids), único y que nunca se reutiliza dentro de una Session. La versión 1 admite un retraso de entero seguro positivo `after_seconds`, un objetivo absoluto explícito `at`, o un intervalo de entero seguro `every_seconds` de al menos cinco minutos. La creación canoniza cada primer objetivo a un `scheduledAt` RFC 3339 UTC de año de cuatro dígitos; un registro `after` conserva su retraso enviado, un registro `at` guarda solo el instante resultante, y un registro `every` conserva su intervalo fijo y su siguiente objetivo.

```ts type-equiv
/** Durable one-shot reminder created from a positive delay. */
interface AfterScheduleRecord {
  /** Session-local stable identity. */
  readonly id: ScheduleId
  /** Rule discriminator for a delayed one-shot reminder. */
  readonly kind: 'after'
  /** Trimmed reminder content supplied at creation. */
  readonly prompt: string
  /** Positive safe-integer delay accepted at creation. */
  readonly afterSeconds: number
  /** Four-digit-year RFC 3339 UTC target. */
  readonly scheduledAt: string
}
```

```ts type-equiv
/** Durable one-shot reminder created from an absolute instant. */
interface AtScheduleRecord {
  /** Session-local stable identity. */
  readonly id: ScheduleId
  /** Rule discriminator for an absolute one-shot reminder. */
  readonly kind: 'at'
  /** Trimmed reminder content supplied at creation. */
  readonly prompt: string
  /** Four-digit-year RFC 3339 UTC target. */
  readonly scheduledAt: string
}
```

```ts type-equiv
/** Durable fixed-rate reminder whose next target remains creation-anchor-aligned. */
interface EveryScheduleRecord {
  /** Session-local stable identity. */
  readonly id: ScheduleId
  /** Rule discriminator for a fixed-rate recurring reminder. */
  readonly kind: 'every'
  /** Trimmed reminder content supplied at creation. */
  readonly prompt: string
  /** Fixed safe-integer interval, never below five minutes. */
  readonly everySeconds: number
  /** Earliest anchor-aligned occurrence not yet dispatched. */
  readonly scheduledAt: string
}
```

```ts type-equiv
/** One-shot record variants that terminate on an id-only dispatch. */
type OneShotScheduleRecord = AfterScheduleRecord | AtScheduleRecord
```

```ts type-equiv
/** The v1 durable reminder record union. */
type ScheduleRecord = OneShotScheduleRecord | EveryScheduleRecord
```

## Entrada de tiempo absoluto

El selector `at` es o una cadena RFC 3339 estricta con offset o un objeto exacto de calendario local. La forma local mantiene explícita su interpretación en el límite de la herramienta:

```ts type-equiv
/** Structured local-calendar input accepted by `schedule_create`. */
interface LocalAtInput {
  /** Four-digit ISO calendar date. */
  readonly date: string
  /** Local wall-clock time with optional one-to-three digit milliseconds. */
  readonly time: string
  /** Explicit UTC or IANA Area/Location zone. */
  readonly time_zone: string
}
```

```ts type-equiv
/** Absolute selector accepted by `schedule_create`. */
type AtInput = string | LocalAtInput
```

El overlay Web oficial muestrea la zona IANA del navegador en cada prompt. El contexto de tiempo le dice al modelo que interprete las fechas y horas en lenguaje natural sin otra calificación en esa zona local a la solicitud cuando el turno abierto tiene una única zona de navegador inequívoca; la procedencia mezclada o ausente le dice al modelo que pregunte. Esa guía no es un valor por defecto duradero de la Session: el modelo debe seguir pasando un offset en la forma de cadena o `time_zone` en la forma local, y Schedule nunca lee el contexto del navegador, de la Session, del proceso ni del modelo.

Schedule rechaza offsets y zonas inválidos, cadenas sin offset, objetivos no futuros y horas locales dentro de huecos de cambio de horario. Un solapamiento de cambio de horario elige su primer instante, el más temprano. La creación exitosa guarda solo el `scheduledAt` UTC canónico, así que la reproducción nunca depende del estado ambiental de zona horaria.

## Entrada de tasa fija y puesta al día

`every_seconds` es un intervalo por registro de al menos 300 segundos, anclado al momento de creación. Es solo recurrencia de tasa fija: el protocolo no tiene expresión de calendario ni Cron, ni zona horaria de recurrencia, ni enfriamiento compartido, ni compuerta de admisión entre registros.

Cuando una Session estuvo fría u ocupada a lo largo de varios objetivos, un registro Every contribuye solo con su ocurrencia debida más reciente. El despacho lo avanza directamente al primer objetivo alineado con el ancla de creación posterior al momento de la decisión de despacho, sin enumerar, persistir ni reproducir los intervalos perdidos. Si ese siguiente objetivo no cabe en un año UTC de cuatro dígitos, el despacho final termina el registro.

Cuando varios registros Every distintos están vencidos y no debe ningún one-shot, cada uno contribuye una ocurrencia al mismo lote de seguimiento en orden de objetivo y de creación. Cada registro Every mantiene estado independiente, mientras que todos los despachos de ese lote admitido usan el mismo momento de decisión. El procesamiento por lotes acota los turnos del modelo; el mínimo de cinco minutos acota la frecuencia del temporizador de cada registro.

## Cambios duraderos y reproducción

El evento de Session `schedule/change` de la versión 1 es la única autoridad duradera de Schedule. Create guarda el registro completo, y delete es una transición terminal solo con id. Un despacho one-shot también es terminal y solo con id. Un despacho Every transporta el momento de decisión de reloj de pared usado para seleccionar su ocurrencia debida más reciente y normalmente avanza el registro activo en lugar de terminarlo. Despachar significa que el seguimiento se encoló de forma síncrona, no que una respuesta del modelo tuviera éxito o que el usuario la leyera.

```ts type-equiv
/** Creates one durable reminder record. */
interface ScheduleCreateChange {
  readonly version: 1
  readonly operation: 'create'
  readonly schedule: ScheduleRecord
}
```

```ts type-equiv
/** Deletes one currently active reminder. */
interface ScheduleDeleteChange {
  readonly version: 1
  readonly operation: 'delete'
  readonly id: ScheduleId
}
```

```ts type-equiv
/** Records that one active one-shot reminder entered the durable dispatch history. */
interface OneShotScheduleDispatchChange {
  readonly version: 1
  readonly operation: 'dispatch'
  readonly id: ScheduleId
}
```

```ts type-equiv
/** Records one fixed-rate decision and advances directly past missed occurrences. */
interface EveryScheduleDispatchChange {
  readonly version: 1
  readonly operation: 'dispatch'
  readonly id: ScheduleId
  /** Wall-clock decision time used to select the latest due occurrence. */
  readonly acceptedAt: string
}
```

```ts type-equiv
/** Durable dispatch shapes supported by the current rule set. */
type ScheduleDispatchChange = OneShotScheduleDispatchChange | EveryScheduleDispatchChange
```

```ts type-equiv
/** Strict version-1 durable Schedule mutation union. */
type ScheduleChange = ScheduleCreateChange | ScheduleDeleteChange | ScheduleDispatchChange
```

El decodificador estricto y el fold rechazan versiones desconocidas, campos extra, ids reutilizados, formas de despacho one-shot o Every no coincidentes, y transiciones de delete o dispatch contra registros inactivos. Una Session normal hace fold de su flujo de eventos completo. Una bifurcación hace fold solo de los eventos desde `SessionHeader.seedLength` en adelante, así que conserva el historial sin adoptar los recordatorios activos de la Session padre. La declaración de `schedule/change` y su ubicación en la fuente también están indexadas en el [catálogo de persistencia](../persistence-catalog.es.md#schedulechange--log-only).

## Vistas activas y gestión

Los valores de las herramientas combinan el registro duradero con el estado de entrega derivado del reloj de pared actual. `session-local` significa que la Session original debe estar en vivo: no existe ningún canal de notificación externo ni planificador de sesiones frías.

```ts type-equiv
/** Current delivery timing derived from the durable record and wall clock. */
type ScheduleState = 'scheduled' | 'overdue'
```

```ts type-equiv
/** Fixed v1 delivery boundary: the original session must be live. */
type ScheduleDeliveryMode = 'session-local'
```

```ts type-equiv
/** Complete model-facing view of one active reminder. */
type ScheduleView = ScheduleRecord & {
  /** Whether the target remains in the future. */
  readonly state: ScheduleState
  /** Reminder delivery never leaves the owning session. */
  readonly deliveryMode: ScheduleDeliveryMode
}
```

El [catálogo de herramientas](../tool-catalog.es.md#deepseek-aidsh-schedule) generado es el dueño de los schemas de argumentos y resultados de `schedule_create`, `schedule_list` y `schedule_delete`. Las llamadas de gestión se serializan con el trabajo debido en una cola con alcance de Agent. Cada lectura o decisión espera primero la barrera de persistencia compartida de la Session; create y un delete real esperan de nuevo después de añadir. Un fallo de barrera informa `persistence_uncertain` en lugar de adivinar si una escritura anticipada llegó a confirmarse. Los otros códigos de error estables son `invalid_prompt`, `invalid_selector`, `invalid_rule`, `invalid_time_zone`, `not_future`, `time_out_of_range`, `frequency_too_high`, `corrupt_schedule_log` e `internal_error`.

## Entrega en vivo

El dueño local al proceso deriva su temporizador más temprano del fold duradero y relee el reloj de pared después de cada espera acotada. Las Sessions frías no hacen ningún trabajo; reabrir una reconstruye los temporizadores y convierte los objetivos pasados en vencidos. Los one-shots debidos tienen prioridad y entran en un turno posterior cada vez. Cuando no debe ningún one-shot, todos los registros Every vencidos forman el lote único descrito antes.

El trabajo debido espera a que el Agent quede totalmente inactivo y reclama la fase de mantenimiento antes de rehacer el fold del estado, muestrear la decisión, encolar un `followup()` y añadir los cambios de despacho correspondientes. Nunca llama a `steer()` ni interrumpe un turno en curso.

El lote one-shot o de tasa fija admitido inicia un turno posterior normal y aparece solo a través del transcript ordinario de la conversación; Schedule no tiene ningún recibo Web duradero independiente ni renderizador de navegador. Si falla el encuadre o la admisión síncrona en la cola, no se registra ningún despacho y el recordatorio sigue activo. El intervalo de crash estrecho entre la admisión y el despacho duradero puede repetir el contenido del recordatorio tras la recuperación, así que la entrega en ese límite es de mejor esfuerzo y al menos una vez, no exactamente una vez.
