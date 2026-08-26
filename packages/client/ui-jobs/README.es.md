# @deepseek-ai/dsh-client-ui-jobs

[English](README.md) | [中文](README.zh.md) | Español

Dueño de la funcionalidad de trabajos en segundo plano web: aporta una entrada a `conversation.session.header.actions` que lista los registros de `ctx.jobs` que esta sesión puede ver. Los datos llegan enteramente a través del espejo de lista `jobsBySession` que [`dsh-client-runtime`](../runtime/README.es.md) deriva de los frames `session/jobs`, así que este paquete no emite ningún RPC y no mantiene estado más allá de la visibilidad del popover.

El disparador se renderiza solo cuando la sesión tiene al menos un trabajo, así que una conversación ordinaria nunca adquiere un control para una capacidad que no está usando. Su insignia cuenta `running` más `stopping` y se omite en cero, dejando a una sesión que solo contiene trabajos terminados una entrada discreta a su historial en lugar de una que anuncia un recuento de nada. El popover es una lista plana: primero las filas vivas por `startedAt` ascendente, luego las filas asentadas por `finishedAt` descendente, con un empate en el mismo milisegundo roto por el orden de inicio para que la iteración del mapa del host nunca lo decida. Una fila muestra el tipo de productor, la etiqueta, un marcador de estado, el `detail` del productor en lugar de la palabra de estado genérica una vez que lo tiene, y una duración transcurrida. Esa duración avanza una vez por segundo mientras la fila está viva y se congela en `finishedAt`; el reloj corre solo mientras una lista abierta contiene algo que se mueve. Una fila asentada a la que le falta `finishedAt` se lee como cero en lugar de como una cifra negativa, y una duración que supera la hora permanece en horas en lugar de crear un vocabulario de días que ningún productor alcanza hoy.

Las filas asentadas permanecen visibles y atenuadas hasta que el registro las suelta al disponer del dueño. Están en la instantánea, el `detail` de un trabajo fallido es el único lugar donde su fallo es legible, y filtrarlas aquí es trabajo que las fases de salida y cancelación desharían. Por eso un subagente en segundo plano de un solo disparo aparece tanto aquí como en el [catálogo de subagentes](../ui-subagent/README.md): el catálogo navega al transcript del hijo, mientras que esta lista es el único asidero al que una cancelación futura puede engancharse.

Escape cierra la lista y devuelve el foco al disparador, igual que una pulsación de puntero fuera de ella. La desaparición del último trabajo cierra la lista antes de que el control se desmonte, así que el foco nunca desaparece de un nodo eliminado. El estilo usa solo tokens; el texto pasa por el espacio de nombres de locale `job` del propio paquete. El comportamiento lo especifica la [Agent Note de visualización de trabajos en segundo plano web](../../../.agents/notes/implemented/feature/2026-08-08-web-background-job-display.md).

## Experiencia de modelo

Ninguna, ya que este paquete renderiza estado de registro calculado por el host para un humano y no toca ningún prompt, mensaje, schema, stream ni resultado de herramienta. La vista del propio modelo de los mismos trabajos permanece en [`dsh-tool-jobs`](../../jobs/tool-jobs/README.md).

#### Efecto en KV Cache

Ninguno; el paquete nunca ensambla ni envía solicitudes de provider.

## Limitaciones conocidas y trabajo diferido

- **Las filas son de solo lectura** — la salida en streaming de un trabajo y una cancelación iniciada por un humano son fases separadas. La cancelación debe además una decisión orientada al modelo que el seam no responde hoy: `kill()` marca la entrega terminal como reportada, así que una interrupción escrita contra el contrato actual dejaría al modelo creyendo que su trabajo sigue en ejecución.
- **La lista no es el conjunto propio del registro** — muestra lo que una sesión puede ver a través de la vista por cable, así que un trabajo propiedad de otra sesión nunca aparece aquí, y un reinicio del proceso vacía la lista mientras el transcript conserva las tarjetas `run_in_background` que iniciaron esos trabajos. Un trabajo sin dueño (uno iniciado sin un `Agent` vivo) es el caso opuesto: llega a la lista de todas las sesiones, coincidiendo con lo que `list(caller)` reporta a cada llamador.
