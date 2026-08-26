# Agent Note: Eventos de ciclo de vida del provider de subagente — `subagent/provider-added` / `subagent/provider-removed`

Status: implemented

[English](2026-07-05-subagent-provider-lifecycle-events.md) | Español

## Problema

[La Agent Note de las variables de prompt](2026-07-05-prompt-variables-and-tool-guidance-ownership.es.md) hace que `dsh-tool-subagent` DERIVE su redactado orientado al modelo de su provider: `SubagentProvider.inheritsParentContext` (spawn/ACP `false`, fork `true`) impulsa tanto la descripción de la herramienta como la descripción del parámetro `prompt`, de modo que la herramienta fork deja de mentir sobre la herencia de contexto. Esa corrección creó una dependencia de datos entre fibras: la descripción de una herramienta se fija en el REGISTRO DE LA HERRAMIENTA (deliberadamente — la descripción es donde vive la guía de elección de herramienta), pero el provider llega en su propia fibra de plugin, sin un calendario particular.

Resolver el provider en el momento de `apply` del plugin de herramientas crea un requisito implícito de orden de carga («lista el backend antes que la herramienta en cordis.yml»). Ese requisito falla porque el Loader de Cordis inicia las entradas hermanas de forma concurrente y `Entry.init()` no espera a la activación: un backend retrasado puede dejar la fibra de la herramienta fallida incluso cuando aparece en primer lugar. El Loader no ofrece ninguna garantía de orden entre hermanos — «async state is not synchronous state» ([patrones defensivos](../../../../docs/defensive-patterns.es.md)).

## Decisión

El registro anuncia la pertenencia de providers como eventos tipados, y el consumidor los refleja en lugar de asumir el orden:

- **`subagent/provider-added(provider)`** — un provider pasó a ser resoluble en el registro `ctx.subagents`. Se emite al registrarse.
- **`subagent/provider-removed(name)`** — un provider abandonó el registro (la fibra de su plugin se dispuso — una descarga o una recarga de HMR (hot module replacement)). Se emite desde el disposer del registro.

`dsh-tool-subagent` refleja el ciclo de vida de su provider nombrado: registra la herramienta cuando el provider está (o pasa a estar) disponible — derivando el redactado de ese provider en ese momento —, anula el registro de la herramienta cuando el provider desaparece y vuelve a derivar el redactado en el re-registro (recarga de HMR). Mientras el provider está ausente la herramienta no existe, lo que no puede mentirle al modelo. Deliberadamente NO queda ningún requisito de orden de carga que documentar: los eventos hacen desaparecer la cuestión del orden en lugar de fijarla.

Los eventos completan también el vocabulario del seam: `ctx.subagents` es un registro nombrado en el que conviven varios backends de delegación (`spawn`, `fork`, `acp`), y un registro del que otros plugins derivan estado debería anunciar los cambios de pertenencia como eventos tipados en lugar de exigir polling o fe en el orden de carga.

## Alternativas consideradas

- **Resolver el provider en el momento de `apply` y lanzar una excepción cuando esté ausente** — rechazada porque «lista primero los backends» reclamaría una garantía de orden del Loader que no existe.
- **Reintentar la búsqueda (hacer polling hasta que el provider aparezca)** — converge al final, pero inventa un protocolo privado de preparación junto al que el framework ya tiene (registro de efectos + disposición); además no puede notar que un provider SE VA, de modo que HMR dejaría varada una herramienta cuyo redactado describe un backend dispuesto.
- **Redactado de subagente solo en sección, resuelto perezosamente en el momento de assemble** — tolera también cualquier orden de carga, pero saca la guía de elección de herramienta de la DESCRIPTION, contradiciendo la regla de propiedad que establece la Agent Note de las variables de prompt (la semántica de cada herramienta y cuándo usarla pertenecen a la descripción). El registro reactivo mantiene la descripción autoritativa Y libre de orden.
- **Clavar el redactado al NOMBRE del provider en lugar de al objeto del provider** — `providerName` es en sí config, así que un provider renombrado recibe en silencio las palabras equivocadas; derivar del `inheritsParentContext` del propio provider resuelto no puede desincronizarse.

## Consecuencias

- Los consumidores que derivan estado de un provider nombrado reaccionan a `subagent/provider-added`/`-removed` en lugar de leer el registro en el momento de `apply`; `dsh-tool-subagent` es la implementación de referencia.
- **La adición falla ruidosamente; la eliminación está contenida por listener.** Un listener de adición puede deshacer el registro. La eliminación se ejecuta durante la disposición, de modo que un listener que lanza se registra en el log sin impedir que los espejos posteriores se ejecuten ni interrumpir el teardown. `start()` sigue resolviendo el provider por nombre en cada ejecución, evitando que las herramientas obsoletas llamen a un backend eliminado. Véanse el [catálogo de eventos](../../../../docs/subsystems/subagent.es.md#cordis-surface) y el [mapa productor/consumidor](../../../../docs/event-producer-consumer.es.md).
- **Una ventana en la que la herramienta está ausente.** Entre la disposición del backend y el re-registro (una recarga de HMR), el modelo no ve ninguna herramienta de subagente. Es el estado honesto — la alternativa es una herramienta que despacha hacia la nada — y la emisión de `tools/change` del registro de herramientas mantiene actualizado el prompt ensamblado.
- **Dos fibras en espera que comparten un `toolName` son una config inválida que se detecta tarde.** Si dos cargas de `dsh-tool-subagent` nombran providers distintos pero el mismo `toolName`, ambas esperan, y quienquiera que llegue primero registra; el segundo registro solo lanza cuando llega SU provider. `TODO(subagent-dup-toolname)` en el plugin registra este radio de explosión; el rechazo de nombres duplicados del registro de herramientas sigue siendo el respaldo.
