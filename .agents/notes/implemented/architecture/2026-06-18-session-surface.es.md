# Agent Note: Session surface — una proyección ordenada sobre el registro de eventos

Status: implemented

[English](2026-06-18-session-surface.md) | Español

## Problema

El registro de eventos es la autoridad, pero la manipulación del historial no tenía ningún mecanismo compartido duradero. Sin él, complementos como la compactación reescribirían las solicitudes derivadas mediante listeners sensibles al orden sin registrar qué eventos usó cada sustitución. Cada manipulación nueva del historial exigiría además cambios en `deriveMessages()`.

## Decisión

Añadir una **surface** — un orden derivado y cacheado de las secuencias de eventos (el subconjunto de eventos que produce mensajes de LLM (modelo de lenguaje de gran tamaño)) — mantenido por marcadores `surfaceOp` en el registro de eventos.

### Dos campos nuevos de nivel superior en `SessionEvent`

Cada `SessionEvent` gana dos campos opcionales (metadatos estructurales, como `seq`/`time`):

- **`sourceEventSeqs?: number[]`** — los números de seq de eventos anteriores citados como fuentes (p. ej., los seqs de `assistant/chunk` que construyeron un `assistant/message`, o los nodos de la surface ensombrecidos por un marcador de compactación). Un `[]` presente solo es válido en `assistant/message` y registra un stream del provider conocido como vacío; cuando el campo está ausente, un evento heredado o ajeno no registra qué eventos anteriores produjeron el mensaje. Otros eventos de surface exigen una lista no vacía cuando el campo está presente. Sin esos seqs citados, la reproducción no puede validar que una operación de reemplazo de rango nombre todos los eventos que eliminó.
- **`surfaceOp?: SurfaceOp`** — cómo entró este evento en la surface. Ausente para eventos que no son de surface.

### SurfaceOp: dos operaciones

```ts
export type SurfaceOp =
  | 'append'                                    // normal tail append
  | { op: 'replace'; start: number; end: number }  // shadow [start, end] inclusive
```

1. **Append (añadir)** — añade el seq del evento nuevo a la cola. Lo usan `user/message`, `assistant/message`, `tool/result`, `context/message`. El loop pasa `surfaceOp: 'append'` en todos esos añadidos y registra `sourceEventSeqs` donde corresponda: cada `assistant/message` correcto registra su conjunto completo de fuentes `assistant/chunk`, incluido `[]`, mientras que `tool/result` registra su fuente `tool/call`.

2. **Replace (reemplazar)** — elimina las entradas desde `start` hasta `end` (ambos inclusive) e inserta el seq del evento nuevo en su lugar. Tanto `start` como `end` deben estar presentes en la surface actual; `start === end` reemplaza una sola entrada. El `sourceEventSeqs` del evento debe contener todos los seqs de surface ensombrecidos. Los eventos ensombrecidos permanecen en el registro pero ya no están en la surface.

### SurfaceManager: basado en deltas, no en reconstrucción completa

Una `Session` posee un `SurfaceManager` que mantiene un `number[]` ordenado de seqs de eventos. El gestor valida cada candidato de semilla o de añadido sin aplicarlo antes de la confirmación, y luego procesa solo los eventos confirmados desde su sincronización anterior en lugar de reescanear todo el registro. `Session.surface` expone el mismo gestor a través del contrato de solo lectura `SessionSurface`, de modo que la aceptación, el historial derivado, la compactación y el contexto del espacio de trabajo comparten un único estado incremental. Replace localiza sus extremos inclusivos por posición en el array e inserta el seq de sustitución en ese rango; ningún segundo gestor, objeto de enlace ni mapa de seq a nodo duplica el orden.

El procesamiento por deltas es O(1) cuando no hay eventos nuevos y O(nuevos eventos) cuando llegan eventos nuevos.

`deriveMessages()` usa la surface cuando existen marcadores de surface, y recurre al recorrido lineal existente para las sesiones sin marcadores (compatibilidad hacia atrás).

### Persistencia

Los campos nuevos se serializan como propiedades JSON de nivel superior. El backend JSONL no requiere ningún cambio — `JSON.stringify`/`JSON.parse` lo preservan todo de forma transparente. La tabla `events` del backend SQLite lleva dos columnas TEXT anulables (`source_event_seqs`, `surface_op`). El `SCHEMA_VERSION` en disco se incrementa para reflejar el conjunto de columnas y, según la política de incrementar-y-rechazar (bump-and-reject) de la fase pre-release, una base de datos escrita por cualquier otra compilación se RECHAZA al abrir en lugar de migrarse (no hay datos de usuario persistidos que actualizar). La `version` del formato de sesión se fija en `SESSION_FORMAT_VERSION = 0` (la postura de «inestable / pre-release»): los campos opcionales de surface se absorben sin incrementarla.

### Recuperación tras un fallo

El módulo `repair.ts` sintetiza cierres de `tool/result` para llamadas de herramienta huérfanas tras un fallo. Esos cierres llevan `surfaceOp: 'append'` y `sourceEventSeqs` que apuntan al evento `tool/call` huérfano, de modo que la surface rehidratada es válida.

### Invariantes

`Session` valida `sourceEventSeqs` y `surfaceOp` en la frontera de semilla/añadido siempre activa: solo `assistant/message` puede usar una lista de eventos de origen vacía; las referencias son únicas, anteriores y conocidas; los extremos del reemplazo existen en el orden de la surface; y `sourceEventSeqs` cubre cada nodo ensombrecido. Son reglas de aceptación de un solo registro y de proyección de almacenamiento, no contribuciones opcionales de servicios de invariantes.

Todo evento elegible para la surface debe llevar `surfaceOp` o desaparecería del historial derivado. Las sobrecargas tipadas de `append` hacen cumplir esto para los tipos de evento literales; las comprobaciones en tiempo de ejecución en `append` y en el constructor de semillas cubren las uniones ampliadas y los registros cargados. Las semillas no válidas se rechazan en lugar de actualizarse, según la política de formato pre-release.

## Alternativas consideradas

- **Envoltorio `agent/request` por complemento** (el patrón previo a la surface para la manipulación del historial) — fragilidad del orden de los listeners, sin registro duradero de lo que se cambió, y cada manipulación nueva fuerza otro cambio en el `deriveMessages()` del núcleo.
- **Rangos de reemplazo semiabiertos `[start, endExclusive)`** — rechazados: los extremos se nombran por seqs de eventos de surface, y el reemplazo de una sola entrada (`start === end`) se lee de forma natural con semántica inclusiva.
- **Objetos de nodo enlazados más un mapa de seq** — rechazados: la producción no leía los enlaces al predecesor, el único uso del sucesor era la siguiente posición del array, y el reemplazo ya exigía una búsqueda lineal con `indexOf`. Un único array de seqs conserva el mismo comportamiento asintótico con una sola representación que validar.
- **Reconstrucción completa detrás de una bandera de sucio** en lugar del procesamiento por deltas — O(N²) a lo largo de la vida de una sesión: cada añadido de un solo evento reescanearía todos los eventos anteriores.

## Consecuencias

- **`packages/core/session`**: `surface.ts` (`SurfaceManager`) mantiene un único array de seqs ordenado para la aceptación de candidatos y la proyección en vivo; `SessionSurface` es su vista pública de solo lectura. `SurfaceOp`/`SurfaceIntent` y los campos de nivel superior del evento de sesión registran cómo se unen las entradas. `append()` exige un `SurfaceIntent` para los eventos de surface, `deriveMessages()` recorre la surface como única vía de derivación, y `repair.ts` emite cierres conscientes de la surface. El constructor de semillas rechaza un evento de semilla elegible para la surface que carezca de su marcador `surfaceOp` (ver § Invariantes).
- **`packages/core/agent-loop`**: todos los añadidos con capacidad de surface pasan las opciones de surface. Cada `assistant/message` cita sus seqs de chunk; cada `tool/result` cita su seq de `tool/call`.
- **`packages/session/session-persistence-sqlite`**: dos columnas TEXT anulables nuevas (`source_event_seqs`, `surface_op`) en la tabla `events`; `SCHEMA_VERSION` incrementado (incrementar-y-rechazar, sin migración).
- **`packages/session/session-persistence-jsonl`**: no se requieren cambios.
- **`packages/session/session-persistence`**: la interfaz abstracta no cambia.

La surface es el cimiento sobre el que se asienta la manipulación del historial: la compactación de dsh-compaction la utiliza de soporte. Un complemento de compactación o de poda de tool-result añade uno de los tipos de evento existentes que producen mensajes (un `user/message` que transporta el resumen, por ejemplo) con `surfaceOp: { op: 'replace', start, end }` y `sourceEventSeqs` que cubren las entradas ensombrecidas — el evento nuevo ocupa el lugar del rango en la surface, mientras que los eventos de rastreo propios del complemento (p. ej. `compaction/start`, `compaction/end`) permanecen fuera de ella. La reproducción conserva la decisión de forma determinista.

Un reemplazo de `tool/result` puede reescribir exactamente un `tool/result` actual y debe conservar todos los campos de datos excepto `content`. La aceptación de `Session` aplica esta regla junto con la validación del rango posicional y de los eventos de origen citados, independientemente de los complementos de diagnóstico opcionales.
