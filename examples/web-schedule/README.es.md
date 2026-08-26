# Schedule local a la sesión

[English](README.md) | Español

Este overlay habilita los recordatorios Schedule en un proceso `dsh web` sin modificar la composición Web predeterminada distribuida:

```sh
dsh web --patch examples/web-schedule/cordis.yml
```

El overlay actual soporta recordatorios creados con un `after_seconds` positivo de número entero, un destino `at` absoluto o un intervalo `every_seconds` de tasa fija de al menos 300 segundos. El modelo los gestiona mediante `schedule_create`, `schedule_list` y `schedule_delete`; todo resultado identifica la entrega como `session-local`.

El navegador adjunta su zona IANA a cada prompt. El contexto de tiempo indica al modelo que interprete las fechas y horas que no llevan calificador en la zona del navegador de esa petición. Esta suposición pertenece solo a la interpretación de lenguaje natural: `schedule_create.at` debe ser o un date-time RFC 3339 estricto con `Z` o un offset numérico, o `{ date, time, time_zone }` con una zona `UTC` explícita o una zona IANA Área/Ubicación. Schedule no retiene ni infiere una zona por defecto de la Session. Los huecos de cambio de hora (daylight-saving) se rechazan, los solapamientos eligen el primer instante y los registros con éxito conservan solo el destino UTC resultante.

El log de la Session original es dueño de cada recordatorio. Un root Agent en vivo espera hasta estar completamente idle y entonces encola un turno de seguimiento normal en esa conversación. Nunca hace steering del trabajo en curso ni añade una tarjeta de recibo o recordatorio aparte. Cerrar el proceso o dejar la Session fría detiene su temporizador en memoria sin borrar el registro; reabrir esa misma Session restaura la espera y entrega un recordatorio vencido. Leer el historial frío nunca lo activa, y un fork no hereda los recordatorios de su parent.

Todos los recordatorios permanecen alineados con su momento de creación. Si uno está vencido, solo se presenta su última ocurrencia vencida y el siguiente destino permanece en la secuencia original de tasa fija. Todos los registros Every distintos vencidos en la misma decisión de idle se combinan en un único seguimiento con una ocurrencia cada uno; los intervalos perdidos no crean un atraso (backlog). Los one-shots vencidos se ejecutan antes de ese lote. Las expresiones Calendar y Cron no se soportan.

Las operaciones de creación y de borrado efectivo confirman el éxito solo después de que la persistencia de la Session confirme su prefijo de evento. Schedule no ofrece notificación por navegador, sistema operativo, correo, SMS ni de otro tipo externo. Un despacho duradero registra que el seguimiento se encoló; no confirma el éxito del modelo ni el recibo del usuario.
