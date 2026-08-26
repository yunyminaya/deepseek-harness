# @deepseek-ai/dsh-command-compact

[English](README.md) | Español

Control `/compact` orientado a humanos sobre [`ctx.compaction`](../compaction/README.es.md). El plugin registra un comando global a través de [`ctx.commands`](../../interaction/commands/README.es.md), de modo que todo adaptador de comando compuesto lo descubre y lo ejecuta sin un turno del modelo. La [Agent Note de compactación manual en cola](../../../.agents/notes/implemented/feature/2026-07-30-queued-manual-compaction.es.md) es dueña de las decisiones de admisión, bloqueo y durabilidad.

## Contrato del comando

| Entrada | Resultado |
|---|---|
| `/compact` | Resume un tramo anterior equilibrado y útil incluso por debajo de la presión automática, y luego informa del recuento de elementos de historial reemplazados y de los tokens estimados después de vaciar el bracket independiente. |
| `/compact` sin historial compactable | `No compactable history yet.` — no se escribe ningún marcador ni mutación de superficie. |
| `/compact <cualquier cosa>` | `Usage: /compact (no arguments)` — el comando no acepta argumentos y no llama a ningún backend de compactación. |

El comando es independiente del backend: depende solo de `compactNow(agent, signal)`. El agent que invoca es el objetivo exacto, y la señal de cancelación de la UI que despacha se reenvía a través del seam. Cada invocación resuelta registra el par `command/run` / `command/done` solo de log, propiedad del executor; ninguno de los dos eventos se une al historial del modelo. En caso de éxito, `command/done.sourceEventSeq` nombra el evento `compaction/summary` de la transacción para que una presentación pueda plegar el ciclo de vida del comando en su checkpoint sin parsear el texto del resultado ni asumir filas adyacentes.

Los códigos esperados de `ManualCompactionError` se convierten en errores directos estables:

| Código | Resultado directo |
|---|---|
| `busy` | `Compaction is unavailable because this process has an active compaction, or the agent is not idle.` |
| `changed` | `The history selected for compaction changed before it could be replaced. The conversation is unchanged; the attempt is recorded in the session log.` |
| `summary` | `Compaction could not produce a useful summary. The conversation is unchanged; the attempt is recorded in the session log.` |
| `commit` | `Compaction did not finish cleanly; some session history may have changed. Inspect the current session state before retrying.` |
| `persistence` | `Compaction finished, but the session could not be saved.` |

El resultado de busy es intencionadamente de ámbito de proceso: un marcador vivo no emparejado bloquea, mientras que un marcador más antiguo que el `session/end-seed` más reciente está obsoleto y no bloquea. Los fallos de implementación inesperados rechazan el despacho. La cancelación sigue siendo autoritativa; el backend completa su limpieza requerida de cierre/vaciado, y el comando se resuelve internamente como `Compaction cancelled.` mientras el executor del comando deja de esperar con su error de cancelación. La disposición del plugin primero desregistra `/compact` y luego drena cada handler que ya haya empezado, de modo que el desmontaje de la raíz no puede cruzar el límite de cierre o vaciado de un comando abortado.

Los prompts enviados mientras corre la compactación siguen aceptándose en el FIFO ordinario del agent con la misma identidad y los mismos hechos de despertar. Solo comienzan después del checkpoint explícito de durabilidad de la compactación y de la liberación de admisión. El contexto inyectado en reposo no se retiene: puede registrarse entre `compaction/start` y `compaction/end`, y el reemplazo posicional lo deja visible después del checkpoint.

## Composición

El producer inyecta `commands` y `compact`. Monta el registro de comandos, un backend y este plugin:

```yaml
- id: commands
  name: '@deepseek-ai/dsh-commands'
- id: compaction-basic
  name: '@deepseek-ai/dsh-compaction-basic'
- id: command-compact
  name: '@deepseek-ai/dsh-command-compact'
```

La base `dsh` distribuida lo monta junto a `compaction-basic`, y el cliente Web proporciona el adaptador de comando. Las superficies de automatización que no componen ningún adaptador de comando conservan solo la compactación automática.

## Experiencia del modelo

### Control humano `/compact`

#### Qué ve el modelo

La entrada de barra y el resultado directo nunca entran en una solicitud al modelo. Una compactación aceptada reemplaza por separado un tramo anterior con el checkpoint de rol de usuario del backend dentro de un bracket independiente `compaction/* { turn: null }`.

#### Efecto en tokens

El ciclo de vida del comando no añade tokens al modelo. Una compactación con éxito reduce las solicitudes posteriores al reemplazar el tramo seleccionado por un resumen enmarcado; el resumen en sí es una solicitud auxiliar.

#### Efecto en la caché KV

El descubrimiento y la contabilidad del comando no afectan a la caché. El reemplazo de superficie aceptado invalida la reutilización desde el primer token de historial ensombrecido.

## Limitaciones conocidas y trabajo pendiente

- **Solo en reposo** — `/compact` informa `busy` cuando un turno o un prompt de despertar ya aceptado tiene prioridad; el comando en sí no se pone en cola.
- **Sin argumentos de rango ni de política** — la forma sin argumentos mantiene el comportamiento estable entre adaptadores de comando. Los rangos explícitos siguen siendo la vía programática `compactRegion()`.
- **Solo adaptadores de comando** — las superficies sin `ctx.commands` no pueden invocarlo y dependen de la compactación automática por presión.
