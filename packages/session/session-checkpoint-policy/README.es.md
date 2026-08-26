# dsh-session-checkpoint-policy

[English](README.md) | Español

Política de durabilidad semántica para agents persistidos. Hace checkpoint de la sesión event-sourced antes de que un adaptador de modelo reciba una solicitud, antes de que el cuerpo de una herramienta de nivel superior pueda producir un efecto secundario externo y en cada frontera `agent/pre-step`, de modo que la respuesta anterior y los resultados ordenados de herramientas sean duraderos antes de la siguiente solicitud.

## Plugin (namespace: `session-checkpoint-policy`)

Este plugin de función sin configuración consume `ctx.sessions`, `ctx.llm`, `ctx.tools` y la presencia de `ctx.sessionPersistence`. Cárgalo junto a un backend de persistencia:

```yaml
- id: session-persistence
  name: '@deepseek-ai/dsh-session-persistence-jsonl'

- id: session-checkpoints
  name: '@deepseek-ai/dsh-session-checkpoint-policy'
```

La persistencia y la programación de checkpoints son deliberadamente plugins de Cordis separados. Un backend de persistencia inicia lotes en segundo plano acotados para las anexiones `session/event` y convierte cada `session/flush` solicitado en una barrera inmediata de quiescence; esta política elige las barreras de solicitud, de despacho de herramientas y de siguiente paso. Cargar un backend sin esta política es válido, pero un fallo puede perder eventos que sigan dentro de la ventana de batching configurada o una escritura pendiente. Las aplicaciones y runtimes persistidos propios montan ambos plugins explícitamente; un despliegue especializado puede omitir o reemplazar la política deliberadamente.

La política envuelve `llm/stream` de forma perezosa, de modo que el stream aguas abajo no se construye hasta que los eventos de solicitud almacenados en buffer de la sesión viva sean duraderos. Envuelve `tools/execute` después de la política y los guards de pre-ejecución; el cuerpo de una herramienta de nivel superior solo se ejecuta después de que su llamada registrada sea duradera. Si la cancelación llega mientras ese flush está pendiente, el wrapper devuelve el resultado canónico `ABORTED_BEFORE_DISPATCH` sin entrar en el cuerpo de la herramienta. Los despachos anidados de herramientas reutilizan el checkpoint de la llamada exterior visible al modelo. `agent/pre-step` persiste el lote anterior de respuesta/resultado antes de la derivación de la solicitud.

El rechazo de un checkpoint falla en cerrado (fail-closed) en las fronteras de modelo y de herramienta: ni el adaptador ni el cuerpo de la herramienta de nivel superior se ejecutan. Un rechazo en la frontera de paso hace fallar el turno antes de que comience otra solicitud. Los checkpoints concurrentes de herramientas comparten el drenaje de persistencia serializado del store de sesión y no pueden duplicar números de secuencia.

## Experiencia de modelo

### Llamadas interrumpidas

#### Lo que ve el modelo

El plugin no añade prompt ni schema de herramientas. Un fallo duro después de un checkpoint de herramienta pero antes de su resultado deja una llamada duradera sin emparejar; la recuperación de sesión suministra el resultado `TOOL_OUTCOME_UNKNOWN` visible al modelo propiedad de `dsh-session`. El mensaje permite reintentar el trabajo de solo lectura o idempotente y exige verificación de estado o confirmación del usuario para las llamadas que puedan tener efectos secundarios.

#### Efecto en tokens

Los checkpoints con éxito no añaden tokens y no cambian la solicitud. La recuperación añade un mensaje corto de resultado de herramienta para equilibrar el transcript interrumpido.

#### Efecto en la caché KV

El resultado de reparación se anexa después del prefijo reutilizable, de modo que no invalida las entradas de caché anteriores.

## Limitaciones conocidas y trabajo diferido

- La política registra de forma duradera la intención de ejecución, no efectos genéricos de exactamente una vez. Las herramientas con efectos secundarios deberían reenviar `exec.callId` como clave de idempotencia cuando su provider soporta una.
- Los eventos de streaming `assistant/chunk` no tienen checkpoint por fragmento. Los lotes en segundo plano acotados normalmente los persisten antes del siguiente checkpoint semántico, pero un fallo duro puede perder el lote en memoria actual o la escritura pendiente.
- Una llamada persistida sin resultado no puede probar si su efecto externo se completó. La recuperación registra por tanto un resultado desconocido en lugar de reintentar automáticamente.
