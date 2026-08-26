# @deepseek-ai/dsh-goal-round-driver

[English](README.md) | Español

Driver de continuación de misma sesión para [`ctx.goals`](../goal/README.es.md). Convierte un objetivo activo y armado en [rondas de objetivo](../../../docs/glossary.es.md#goal-round) secuenciales a través de los servicios públicos `Agent` y de sesión; la [Agent Note del driver de misma sesión](../../../.agents/notes/implemented/feature/2026-07-19-same-session-goal-round-driver.es.md) es la dueña del razonamiento sobre carreras y ciclo de vida.

## Composición

```yaml
- id: goal
  name: '@deepseek-ai/dsh-goal'

- id: tool-goal
  name: '@deepseek-ai/dsh-tool-goal'

- id: goal-round-driver
  name: '@deepseek-ai/dsh-goal-round-driver'
```

El plugin no tiene configuración ajustable. `maxGoalRounds` pertenece a la definición de objetivo, mientras que el umbral de bloqueo orientado al modelo pertenece a [`dsh-tool-goal`](../tool-goal/README.es.md); duplicar cualquiera de los dos valores en el driver podría producir políticas divergentes.

## Contrato de ronda

Cuando un agent en vivo exacto está inactivo con un objetivo activo y armado y capacidad restante, el driver primero registra un punto de control de las mutaciones de objetivo pendientes y después reserva `roundsStarted + 1` para el `{ goalId, revision }` actual. Pone en cola un prompt `<goal_round>` con `GoalMessageSource`. El listener de `agent/pre-step` verifica el registro reclamado completo y el objetivo actual tanto antes como después de los listeners posteriores; solo un `user/message` entrado incrementa `roundsStarted`. Una reserva rechazada como obsoleta no consume el número de ronda.

`MessageId` identifica el mensaje reservado mediante la inserción y la reclamación duraderas en la bandeja de entrada; no identifica un resultado de turno. Los mensajes humanos no consumen el tope de objetivo. Si el trabajo humano entra en la bandeja antes de una reserva o se une a su lote pendiente, el trabajo automático cede hasta que el agent queda inactivo; un prompt automático pendiente en un lote mixto se rechaza y solo se vuelve a reservar después de ese punto de control.

El prompt retenido nombra el objetivo entre comillas JSON y `round/maxGoalRounds`, trata el espacio de trabajo actual, los resultados de herramienta y el estado de sesión duradero como autoritativos, exige evidencia antes de completar y dice al modelo que deje el objetivo activo cuando quede trabajo. Las comillas conservan el texto de objetivo multilínea o con forma de etiqueta como datos. Las mutaciones del ciclo de vida de objetivo siguen exigiendo las comprobaciones de autoridad independientes de `dsh-tool-goal`.

## Punto de control de inactividad

En la inactividad completa del agent, la fase y la revisión duraderas del objetivo son autoritativas. Un objetivo activo y armado con capacidad reserva su siguiente ronda; la finalización, la pausa, el bloqueo y las ediciones suprimen la continuación. El driver no clasifica la actividad precedente correlacionando el mensaje de objetivo con `turn/end`, así que los errores de provider y los límites de tokens no son resultados de objetivo a nivel de prompt.

## Ciclo de vida y durabilidad

`goal/changed` crea una obligación de durabilidad. Antes de poner trabajo en cola, el driver espera `ctx.sessions.flush()` y, después de la espera, vuelve a comprobar tanto la revisión del objetivo como la entrada competidora. Un fallo de flush que llega por `agent/error` desarma la continuación antes de que pueda empezar otra ronda.

La activación nunca se hereda cuando este plugin se carga sobre un agent existente. `GoalService.disarm()` elimina la autoridad local del proceso sin cambiar la fase, la revisión ni el historial duraderos; el resume explícito autorizado por humano registra la reactivación posterior. La misma regla se aplica después del resume y del fork de sesión mediante el manejo de `agent/session-start` del dominio de objetivo.

La cancelación elimina el trabajo pendiente de la bandeja de entrada o deja un estado abortado de todo el agent. En el siguiente punto de control de inactividad, el driver pausa un objetivo con un intento reservado o admitido para que la cancelación no pueda reiniciarlo automáticamente; la cancelación no relacionada con un intento de objetivo solo desarma la continuación local del proceso. Si la mutación de pausa falla, el driver recurre a desarmar. El desmontaje del plugin cierra la admisión, desarma cada objetivo en vivo, cancela el trabajo activo con la causa `parent` y espera la quietud del driver y del agent mientras su barrera de eventos permanece instalada.

## Experiencia de modelo

### Prompt de ronda de objetivo

#### Lo que ve el modelo

Cada ronda admitida es un bloque `<goal_round>` retenido con rol de usuario que nombra el objetivo completo y el número de ronda positivo. Los mensajes humanos anteriores, las instantáneas de estado de objetivo, la salida del asistente y los registros de herramienta permanecen en el mismo historial de sesión.

#### Efecto en tokens

Por cada ronda admitida se añade un bloque fijo de instrucciones más el objetivo. Las peticiones posteriores reenvían las rondas retenidas hasta que la compactación las oculta; no se crea ningún agent nuevo ni prefijo de conversación copiado.

#### Efecto en la KV cache

Solo añadidura dentro de una época: cada ronda admitida extiende la conversación existente después de su prefijo reutilizable. La compactación puede reemplazar el sufijo de historial derivado y desplazar el límite reutilizable.

## Limitaciones conocidas y trabajo pendiente

- **Sin evaluador independiente** — la política de objetivo orientada al modelo decide cuándo la evidencia es suficiente para completar y si un bloqueador no ha cambiado semánticamente; la certificación respaldada por un evaluador sigue pendiente.
- **Solo ejecución de misma sesión** — este paquete deliberadamente no hace spawn de un agent nuevo, no bifurca un prefijo de sesión ni implementa intentos independientes al estilo Ralph; ese flujo de trabajo pertenece a su propia capa de plugins.
- **Carrera de descarga de la cola aceptada** — la descarga de plugins de Cordis es asíncrona. Un prompt de objetivo ya aceptado por la bandeja del agent puede comenzar y consumir su ronda antes de que empiece la descarga; el desmontaje entonces cancela la petición, desarma el objetivo y espera la quietud. No empieza ninguna ronda posterior.
- **Tope de rondas, no presupuesto de recursos** — las políticas de tokens, moneda, tiempo y cuota de provider siguen siendo independientes. Sus eventos de sesión no se atribuyen al mensaje de objetivo ni se asignan a códigos de bloqueador de objetivo.
- **Sin reintento automático ante anomalías** — los fallos transitorios de provider y de persistencia exigen un resume posterior autorizado por humano en lugar de una política de reintento implícita.
