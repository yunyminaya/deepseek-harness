# @deepseek-ai/dsh-subagent-spawn-in-process

[English](README.md) | Español

El provider de spawn crea un `Agent` hijo nuevo en el proceso actual. El hijo tiene su propia sesión, no ve el historial de conversación del padre y reutiliza la fábrica de agents del host y los servicios de LLM/herramientas.

## Comportamiento

`start(request)` delega en [`startInProcessRun`](../subagent-in-process-driver/README.es.md) sin semilla y espera la publicación antes de devolver. El hijo recibe el linaje de directorio de trabajo/sesión del padre y hereda el modelo del padre salvo que se anule, pero arranca con una conversación vacía.

El driver compartido es dueño de la comprobación de profundidad, la configuración de persona y filtro de herramientas, la salida estructurada, la cancelación por señal requerida, la ejecución one-shot, la lectura del resultado y la liberación en quiescencia. Un rechazo de arranque no deja ningún hijo publicado; la descarga del provider tras el cumplimiento no revoca la ejecución propiedad del titular.

## Capacidades

Spawn anuncia `{ outputSchema: true, depthLimit: true, toolFilter: true, persona: true }` porque controla la ventana de creación del hijo y puede aplicar las cuatro funcionalidades.

## Configuración

| Clave | Significado |
|---|---|
| `providerName` | Nombre de registro en `ctx.subagents` (por defecto `spawn`). |

## Experiencia del modelo

### Solicitud de agent hijo

#### Lo que ve el modelo

El hijo nuevo recibe el contenido de la tarea autónoma verbatim, hereda por defecto el modelo y el workspace del padre, y ve el prompt global con cualquier sombra de persona con ámbito de hijo configurada. Un filtro de herramientas elimina los schemas de wire globales, la búsqueda de ejecutables y los enlaces de SDK de Code Mode para ese hijo, pero deja la guía registrada de forma independiente. No recibe ningún mensaje de conversación del padre; el filtro es visibilidad/composición, no una concesión de autoridad heredada del padre.

#### Efecto en tokens

El hijo paga un contexto y un historial nuevos e independientes; no se duplican tokens del historial del padre. La persona cambia el coste repetido de prompt de este hijo, mientras que el filtrado cambia su coste de schema o de SDK generado.

#### Efecto en KV Cache

Independiente de la caché de solicitudes del padre. El historial del hijo crece de solo añadido, mientras que los cambios de persona, filtro de herramientas, SDK generado, provider o modelo establecen un prefijo de hijo distinto.

### Resultado de herramienta del padre, indirectamente

#### Lo que ve el modelo

A través de `dsh-tool-subagent`, el padre recibe solo la salida final del hijo o el error de motivo de parada.

#### Efecto en tokens

La entrada del padre crece con un resultado dependiente de los datos retenido hasta la compactación.

#### Efecto en KV Cache

De solo añadido; el contenido recién visible sigue el prefijo de solicitud reutilizable y no invalida las entradas existentes de la caché KV.

## Limitaciones conocidas y trabajo diferido

- **Nuevo significa sin transcript del padre** — el hijo hereda cwd, linaje, modelo y las restricciones explícitas de persona/herramientas, pero nada de la conversación del padre; usa el provider fork cuando se requiera el contexto de turnos completados.
