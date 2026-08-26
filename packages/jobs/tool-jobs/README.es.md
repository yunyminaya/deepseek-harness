# @deepseek-ai/dsh-tool-jobs

[English](README.md) | Español

El controlador orientado al modelo para `ctx.jobs`: tres herramientas independientes del tipo, avisos de finalización y una sección de prompt para el trabajo en segundo plano. Cargar el plugin adjunta el controlador que requiere `ctx.jobs.start()`.

## Herramientas

- `job_output(job_id, wait?, timeout_ms?)` lee sin bloquear por defecto. Los trabajos en streaming devuelven solo el siguiente delta; los trabajos de salida final devuelven su resultado tras la liquidación. Toda respuesta termina con `[status: ...]`. `wait: true` espera hasta el tope configurado y, al agotarse el tiempo, deja vivo un trabajo que sigue en ejecución.
- `job_list()` devuelve los trabajos visibles para el llamador como `<id> [<kind>] <status> — <label>`.
- `job_kill(job_id, reason?)` solicita la cancelación de inmediato y reenvía el motivo registrado. Los trabajos terminales devuelven una instantánea no consumidora.

Las tres usan tarjetas de UI genéricas: `read` para la salida y el listado, `execute` para el kill.

Sus valores canónicos son `{ text, job }`, `PublicJobSnapshot[]` y `{ outcome: 'cancellation-requested' | 'already-finished', job }`. Una instantánea pública lleva id, kind, label, status/detail y las horas de inicio/fin; omite deliberadamente `ownerSession` y el bit interno `reported` del aviso. Los renderizadores nativos conservan el estado y el texto de acuse antes mencionado.

Cuando un productor aporta `outputLimitBytes`, `job_output`, el `job_kill` terminal y los avisos de finalización acotan el resultado UTF-8 nativo completo después de añadir el texto de estado o de aviso. Las lecturas conservan la cola de salida y el sufijo de control cuando caben; un aviso de finalización acotado reserva en cambio `background job <id>` y la instrucción de recolección de `job_output` antes de gastar los bytes restantes en su kind, label, status, detail y marcador de truncamiento variables. Un listener previo a la ejecución antepuesto captura el trabajo visible para el llamador antes de la política, y el callback de contenido final de cada definición de control de trabajo aplica el tope del productor a las denegaciones de texto único, los cortocircuitos, los fallos normalizados de herramienta o pipeline, los reemplazos y los bloqueos; los resultados de política estructurados de varios bloques conservan su forma. Un marcador de truncamiento del productor ya existente se reutiliza en lugar de duplicarse. Los productores que omiten el campo conservan el comportamiento existente del controlador sin acotar.

## Avisos de finalización

Una finalización no informada entrega `background job <id> (<kind>: <label>) finished [status: ...]. Read its output with job_output.` al propietario exacto. Cuando está acotado, el prefijo de id estable y el comando de recolección tienen prioridad sobre el label/detail variables para que el aviso siga siendo accionable en el mínimo de 64 bytes admitido por la PTY. Un kill o una lectura/espera terminal marcan la entrega como informada y suprimen el aviso redundante, igual que la cancelación de teardown que drena a un propietario o al servicio.

Qué carril la transporta depende de lo que esté haciendo el propietario. A un propietario ocupado se le inyecta: el aviso entra en la bandeja de entrada de siguiente paso y el turno no puede cerrarse mientras esa bandeja lo retenga, de modo que varios trabajos que se liquidan a la vez cuestan un paso en lugar de un turno cada uno. A un propietario inactivo se le despierta en cambio con un turno de seguimiento, porque un aviso pendiente que nada reclama es una finalización de la que el modelo nunca llega a enterarse. `completionDelivery: quiet` conserva el carril de inyección también para los propietarios inactivos, que es lo que necesita un transcript determinista.

El despertar está acotado. Cada propietario puede abrir `maxConsecutiveWakes` turnos de este modo antes de que los avisos posteriores se degraden a inyección, y reclamar cualquier mensaje redactado por el usuario restaura el presupuesto. El límite existe porque la cadena es autoexcitante: un turno despertado puede iniciar el trabajo en segundo plano cuya finalización lo despierta de nuevo. Los avisos que este plugin puso en cola nunca rellenan el presupuesto que gastaron.

Un registro host puede portar varios montajes de este plugin — uno por preset de agent. El registro enruta cada liquidación a los listeners que alcanza la cadena de ámbitos del propietario, de modo que un montaje bajo un preset nunca ve los agents de otro preset y un agent lee exactamente un aviso por finalización, sin importar cuántos presets estén montados. El mismo enrutado decide a qué agents sirve el controlador de este montaje: un agent cuya composición no carga `tool-jobs` no puede iniciar trabajo en segundo plano en absoluto.

## Configuración

| clave | por defecto | significado |
|---|---|---|
| `waitTimeoutMs` | `30000` | espera usada cuando `wait: true` omite `timeout_ms` |
| `maxWaitTimeoutMs` | `600000` | tope para las esperas suministradas por el modelo |
| `completionDelivery` | `wakeup` | `wakeup` abre un turno en un propietario inactivo; `quiet` deja el aviso pendiente |
| `maxConsecutiveWakes` | `3` | turnos que un propietario puede abrir por despertar antes de que los avisos se degraden a inyección |

Un valor por defecto por encima del tope falla al cargar.

## Experiencia del modelo

### Prompt de sistema

#### Lo que ve el modelo

Toda solicitud en el ámbito de registro de este plugin contiene esta guía. El filtrado de herramientas con ámbito de agent puede ocultar las herramientas sin eliminar la sección de prompt registrada de forma independiente.

##### Guía de trabajo en segundo plano

```markdown
Track every background job id you start. You are notified in-session when a job finishes — do not busy-poll or sleep on one; keep working on independent steps and do not duplicate a running job's work. Before giving a final answer, collect every still-relevant job with job_output (set wait: true only when you are genuinely blocked on it), and job_kill jobs that stopped mattering.
```

#### Efecto de tokens

Coste de entrada pequeño y fijo por solicitud mientras está activa.

#### Efecto de KV Cache

Estable de prefijo mientras el ámbito del plugin y el texto de la guía no cambien. La activación o la disposición pueden invalidar la reutilización de esta sección de prompt.

### Schemas de herramientas

#### Lo que ve el modelo

Los schemas generados de [`job_output`, `job_list` y `job_kill`](../../../docs/tool-catalog.es.md#deepseek-aidsh-tool-jobs) mientras este conjunto de herramientas sea visible.

#### Efecto de tokens

Coste de schema fijo en cada solicitud donde las herramientas sean visibles.

#### Efecto de KV Cache

Estable de prefijo mientras las definiciones de herramientas y la visibilidad no cambien. El ciclo de vida del registro o las restricciones con ámbito pueden invalidar la reutilización desde el primer token de schema cambiado.

### Resultados y avisos

#### Lo que ve el modelo

Las lecturas devuelven la salida o `(no new output)` seguida de `[status: <status>]` y un detail opcional. Una lista vacía devuelve `(no background jobs)`. El kill devuelve `requested cancellation of job <id>` o el estado terminal existente. Una finalización propia no informada usa el aviso anterior.

#### Efecto de tokens

Los resultados y los avisos permanecen en el historial del padre hasta la compactación. Las lecturas de streaming no repiten la salida ya consumida; un `outputLimitBytes` suministrado por el productor acota cada lectura o aviso completo. Con `wakeup`, un aviso que llega a un propietario inactivo compra además una solicitud de modelo que el usuario no pidió, limitada por propietario mediante `maxConsecutiveWakes`; un aviso que llega a un propietario ocupado añade un paso al turno que ya está pagando.

#### Efecto de KV Cache

De solo anexión; el contenido recién visible sigue el prefijo de solicitud reutilizable y no invalida las entradas de KV Cache existentes.

## Limitaciones conocidas y trabajo diferido

- **Una liquidación dentro de la ventana de retiro del driver sigue dejando su aviso varado** — entre la última comprobación de la bandeja de entrada del loop de turnos y el momento en que el driver confirma su fase de inactividad, el propietario sigue leyéndose como ocupado, así que el aviso se inyecta y nada lo despierta. El steering tiene la misma laguna; cerrarla corresponde a `agent-loop`.
- **Un presupuesto de despertar gastado no se restaura con el tiempo** — solo la entrada redactada por el usuario lo rellena, así que un agent desatendido cuyo presupuesto se agotó recoge sus avisos restantes en el siguiente turno que otra cosa abra.
- **Un aviso pendiente en un propietario inactivo no sobrevive a la disposición de ese propietario** — la cancelación de disposición limpia la bandeja de entrada no reclamada, y el log conserva el par insertar/cancelar como registro.
- **Las lecturas de streaming son de un solo consumidor** — los observadores independientes necesitan otra API de runtime.
- **Los trabajos sin propietario no tienen valla de sesión** — los llamadores externos deben aportar una política o evitarlos.
