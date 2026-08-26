# Agent Note: Contexto de tiempo duradero por paso

Status: implemented

[English](2026-07-16-durable-per-step-time-context.md) | Español

## Problema

Un reloj solo-de-petición puede decirle al modelo la hora actual, pero sustituir ese valor en el system prompt elimina la evidencia detrás de razonamientos sensibles al tiempo anteriores. Los turnos multi-paso necesitan que las peticiones conserven las lecturas usadas por los pasos precedentes. La petición debe seguir siendo reconstruible después de un reinicio, y la compactación automática debe dar cuenta del mismo contexto de tiempo que recibe el modelo.

Una caché de refresco local al proceso hace que el tiempo mostrado dependa de un estado que no puede sobrevivir a la reanudación. El lenguaje natural originado en el navegador también necesita una zona propiedad de la petición: una zona de proceso de servidor no puede inferir la localidad del usuario, mientras que un valor por defecto mutable de Session o de conexión deja que los viajes o las pestañas concurrentes reinterpretan otro prompt.

## Decisión

`@deepseek-ai/dsh-time-context` es un plugin de función opt-in en `packages/context/time-context/`. Las composiciones por defecto dejan desactivados su divulgación y su coste de tokens; el overlay de Schedule Web lo monta para que el modelo pueda interpretar fechas y horas sin calificar en la zona del navegador adjunta a la petición actual.

El plugin antepone un listener de `agent/pre-step` y delega primero. Cuando la decisión descendente entra y toca una lectura, combina los mensajes finales de esa decisión con los mensajes de usuario duraderos ya presentes en el turno abierto, deriva la procedencia de la zona del navegador a partir de fuentes exactas `user-rpc` y añade una lectura a la decisión. El rechazo, el fallo del listener o una señal ya abortada no registran nada. El steering reclamado después del lote actual conserva la propiedad ordinaria del siguiente paso y recibe una lectura nueva cuando ese paso entra.

Cada prompt Web muestrea la zona IANA del navegador. El Host la valida y canoniza antes de vincularla a la fuente exacta del mensaje de usuario duradero. Una zona única en el turno abierto resuelve la petición; varias zonas producen un resultado `mixed` ordenado; ninguna zona es `unavailable`. Una petición resuelta le dice al modelo que interprete las fechas y horas sin calificar en esa zona. Una procedencia mixta o no disponible le dice que pida al usuario que aclare.

Esta procedencia vinculada al mensaje no se copia a `SessionHeader`, a un valor por defecto de conexión ni al estado de Schedule. Time-context posee solo la guía al modelo. Una herramienta que acepte campos de calendario locales debe seguir fijando su propia frontera explícita; por eso Schedule exige `time_zone` en lugar de importar la lectura de este plugin ([decisión](../simplification/2026-08-09-explicit-schedule-time-zone.es.md)).

La zona resuelta del navegador formatea también la marca de tiempo de la lectura. Las peticiones mixtas o no disponibles usan el respaldo `timeZone` configurado, o la zona del proceso Node resuelta una vez en la carga del plugin cuando se omite la configuración, conservando la política de aclaración. Cada respaldo se valida a través de `Intl.DateTimeFormat`.

Cada lectura usa la fuente de instantánea exacta `{ kind: 'plugin', plugin: 'time-context', form: 'snapshot', sections: [{ name: 'time-context', text: <same text> }] }`. El companion de invariantes comprueba la forma de la instantánea, re-deriva la procedencia de navegador del turno actual a partir de los mensajes `user-rpc` originales y valida la zona de la marca de tiempo renderizada y la línea base de tiempo transcurrido.

La configuración opcional `refreshIntervalMs` es un entero seguro no negativo. La omisión o `0` inyecta en cada paso entrado elegible. Un valor positivo escanea los eventos Session crudos buscando la lectura de plugin más reciente e inyecta cuando no existe ninguna, el tiempo de pared retrocedió o el evento tiene la antigüedad suficiente. La marca de tiempo del evento gobierna después de la compactación y la reanudación sin una caché local al proceso. El overlay de Schedule Web omite el intervalo para que cada paso de petición reciba guía de navegador actual.

### Texto y líneas base de tiempo transcurrido

Una lectura resuelta del primer paso es:

```text
Time sampled while preparing turn <turn>, step 1: <timestamp-in-browser-zone>
Browser time zone for this request: <iana-zone>. Interpret otherwise-unqualified dates and times in this zone.
Elapsed since the preceding model-visible message: <duration-or-unavailable>.
```

Las variantes mixta y no disponible sustituyen la segunda línea por una instrucción de pedir aclaración. La línea base es el último mensaje duradero precedente de usuario, asistente o resultado de herramienta. El prompt propuesto para este paso aún no se ha añadido; una Session nueva puede por tanto informar `unavailable`.

Una lectura de un paso posterior cambia el número de paso de la primera línea y termina con:

```text
Elapsed since the preceding step context: <duration-or-unavailable>.
```

Esa línea base es el evento de contexto de tiempo precedente en el turno abierto. Las líneas base ausentes informan `unavailable`; el formateo de duración usa unidades compactas de segundos enteros y sujeta a cero el retroceso del reloj de pared.

### Durabilidad y reconstrucción

Un paso entrado añade sus mensajes devueltos seguidos de la lectura de tiempo después de `step/start`, antes de la derivación de la petición. Un fallo posterior de preparación puede dejar la lectura en el historial porque registra la entrada, no la transmisión exitosa. Cada lectura sigue siendo un nodo de surface normal hasta que la compactación la sombrea. Un intervalo positivo puede dejar que una petición posterior reutilice el historial existente sin añadir una lectura nueva.

El plugin no contribuye nada al ensamblaje del system prompt ni a `request/header`. La reconstrucción de peticiones obtiene el prefijo completo de surface duradera en cada `step/start`, de modo que las peticiones históricas recuperan el tiempo exacto y la política de navegador que vio el modelo.

## Alternativas consideradas

- **Sustituir un valor dinámico del system prompt** — rechazado porque la sustitución borra las lecturas previas y cambia las peticiones históricas reconstruidas.
- **Persistir una zona por defecto de Session** — rechazado porque el hecho del navegador pertenece a un prompt; los viajes y las pestañas concurrentes no deben mutar el significado compartido ni propagar estado de zona a través de los contratos de Session, fork y persistencia.
- **Copiar la zona del navegador a una segunda autoridad de contexto** — rechazado porque la fuente `user-rpc` original ya la posee y el invariante puede re-derivar la política directamente.
- **Dejar que Schedule consuma la lectura implícitamente** — rechazado porque el contexto en prosa no es un valor por defecto tipado estable y acoplaría un analizador de tiempo absoluto al historial de AgentLoop. El modelo pasa en su lugar un offset o una zona explícitos.
- **Usar solo la zona del proceso** — rechazado porque la localidad del despliegue no puede inferir la zona de un usuario remoto. Sigue siendo un respaldo de visualización cuando la procedencia de la petición está ausente o es mixta.
- **Exponer el tiempo solo a través de una herramienta** — rechazado porque el razonamiento temporal ordinario exigiría un viaje de ida y vuelta evitable y no aseguraría una lectura antes de cada paso.
- **Montar time-context por defecto** — rechazado porque la divulgación, la frescura y el coste de historial siguen siendo política de composición.

## Verificación

Las pruebas unit y de loop real fijan el formateo de marcas de tiempo, la derivación de navegador única/mixta/ausente, el respaldo de visualización, ambas líneas base de tiempo transcurrido, los límites de intervalo, la programación entre turnos y reanudada, el comportamiento de reloj hacia atrás, la propiedad del steering, la cancelación, la validación exacta de instantáneas y la reconstrucción de peticiones. Las pruebas de Host/cliente fijan el muestreo del navegador más la validación y canonización en la entrada del prompt. El escenario ensamblado sin clave de Schedule Web envía un prompt de navegador real, observa la misma zona en la petición del modelo y verifica que el modelo la suministra explícitamente a `schedule_create`.

## Consecuencias

- El significado de la zona del navegador es local a la petición y duradero sin cambiar los schemas de Session, fork, JSONL o SQLite.
- El modelo recibe la suposición local al navegador solicitada en cada paso de petición de Schedule Web; la procedencia mixta o ausente pregunta en lugar de adivinar.
- Las herramientas siguen siendo explícitas: el contexto ayuda al modelo a elegir campos pero no se convierte en un valor por defecto oculto del seam de paquetes.
- El contexto de tiempo permanece solo-añadido hasta la compactación; un intervalo positivo reduce el crecimiento del historial pero puede omitir guía de navegador fresca en peticiones posteriores.
