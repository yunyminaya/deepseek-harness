# @deepseek-ai/dsh-time-context

[English](README.md) | Español

Contexto duradero opcional con la hora actual en la zona configurada, la zona de navegador adjunta a la petición abierta y el tiempo transcurrido muestreado durante la preparación de la petición del modelo. Las composiciones por defecto lo dejan desactivado; el overlay Schedule Web lo monta para que el modelo pueda interpretar fechas y horas no cualificadas en la zona de navegador del usuario. Registro de decisión: [la Agent Note del time-context duradero](../../../.agents/notes/implemented/feature/2026-07-16-durable-per-step-time-context.es.md).

## Config

```yaml
- id: time-context
  name: '@deepseek-ai/dsh-time-context'
  config:
    timeZone: Asia/Shanghai  # optional fallback when the request has no unique browser zone
    refreshIntervalMs: 60000 # optional; omit or set to 0 for every eligible attempt
```

Cuando el turno abierto contiene una zona de navegador validada por el Host, esa zona local a la petición formatea la marca de tiempo. Con procedencia de navegador ausente o mixta, `timeZone` suministra el respaldo de visualización; omitirlo resuelve la zona del proceso Node una vez en la carga del plugin. Node honra `TZ`, y todo respaldo explícito se valida mediante `Intl.DateTimeFormat`.

`refreshIntervalMs` debe ser un entero seguro no negativo. La omisión o `0` añade contexto a cada pre-step entrante elegible cuya señal no esté ya abortada. Un valor positivo lo añade solo cuando la Session no tiene ninguna inyección de time-context anterior, el tiempo de pared retrocedió, o han transcurrido al menos esos milisegundos desde la última inyección.

## Propiedad de la zona de petición

El navegador muestrea `Intl.DateTimeFormat().resolvedOptions().timeZone` para cada prompt. El Host valida y canoniza ese valor antes de vincularlo a la fuente exacta del mensaje duradero `user-rpc`. Time-context examina solo esas fuentes en el turno abierto: una zona única resuelve la petición, varias zonas son `mixed` y ninguna es `unavailable`. No lee ni muta los headers de Session, el estado de conexión ni los registros de Schedule.

La instrucción resuelta dice al modelo que interprete las fechas y horas no cualificadas en esa zona de navegador. La procedencia mixta o no disponible dice al modelo que pida al usuario que aclare. Esto es contexto de lenguaje natural, no un valor por defecto de entrada en otro límite de paquete: una herramienta que acepta campos de calendario locales sigue siendo dueña de su requisito explícito de zona.

## Semántica de temporización

El plugin antepone un listener de `agent/pre-step` y delega primero. Cuando una inyección debe ocurrir y la decisión descendente entra, añade un `UserMessage` con fuente al lote devuelto. AgentLoop registra el lote final después de `step/start` y antes de la derivación de la petición. El rechazo, el fallo del listener o una señal ya abortada no registran nada.

Cada lectura usa la fuente de instantánea exacta `{ kind: 'plugin', plugin: 'time-context', form: 'snapshot', sections: [{ name: 'time-context', text: <same text> }] }`. El compañero `./invariant` valida esa forma, re-deriva la política de navegador del turno actual desde los mensajes `user-rpc` originales y comprueba la zona de la marca de tiempo y la línea base transcurrida.

La programación de intervalo positivo escanea los eventos duraderos en bruto de Session en busca del mensaje más reciente atribuido al plugin, incluida una lectura ensombrecida por compactación. Sobrevive por tanto a la reanudación sin caché local al proceso. Un intervalo positivo puede dejar deliberadamente que una petición posterior reutilice el historial existente sin una lectura nueva; el overlay Schedule Web omite el intervalo.

El paso 1 mide desde el último mensaje duradero precedente de usuario, asistente o resultado de herramienta. El prompt propuesto para ese paso aún no se ha añadido. Los pasos posteriores miden desde el evento de time-context precedente en el mismo turno. Las líneas base ausentes reportan `unavailable`, y el movimiento hacia atrás del reloj de pared recorta a cero el tiempo transcurrido.

Una lectura registra un paso entrado, no una petición completada o transmitida. Un fallo posterior de preparación puede dejarla en el historial. El mensaje permanece en el historial de conversación derivado hasta que la compactación lo ensombrece; `request/header` no contiene estado de time-context, y la reconstrucción de la petición usa el prefijo completo de superficie duradera después de cada `step/start`.

## Model Experience

### Contexto temporal de preparación

#### Lo que ve el modelo

Cada mensaje inyectado contiene tres líneas. `<timestamp>` es una marca de tiempo con forma ISO, offset numérico y zona IANA; las duraciones usan unidades compactas de segundos enteros.

##### Primer paso

```markdown
Time sampled while preparing turn <turn>, step 1: <timestamp>
Browser time zone for this request: <iana-zone-or-mixed-or-unavailable-policy>.
Elapsed since the preceding model-visible message: <duration-or-unavailable>.
```

##### Pasos posteriores

```markdown
Time sampled while preparing turn <turn>, step <step>: <timestamp>
Browser time zone for this request: <iana-zone-or-mixed-or-unavailable-policy>.
Elapsed since the preceding step context: <duration-or-unavailable>.
```

#### Efecto de tokens

Cada lectura se acumula hasta que la compactación la ensombrece. Un intervalo positivo reduce las adiciones; la omisión o `0` añade una en cada intento de preparación elegible.

#### Efecto de caché KV

Solo añadidura; el contenido recién visible sigue el prefijo de petición reutilizable y no invalida las entradas de caché KV existentes.

## Limitaciones conocidas y trabajo diferido

- **Solo procedencia de prompt** — el contexto de zona de navegador guía la interpretación en lenguaje natural pero no suministra silenciosamente el campo de zona requerido por otra herramienta.
- **Los turnos mixtos preguntan** — si un turno abierto contiene prompts de diferentes zonas de navegador, se dice al modelo que aclare en lugar de adivinar cuál es dueña de un tiempo no cualificado.
- **El respaldo no es autoridad de usuario** — la zona configurada o del proceso formatea el reloj cuando la procedencia de navegador falta o es mixta, pero la política orientada al modelo sigue diciendo que se aclare.
- **Visualización de segundos enteros** — las marcas de tiempo y las duraciones omiten la precisión de subsegundo aunque los tiempos de evento duraderos retienen milisegundos.
- **Coste de historial entre compactaciones** — la omisión o `0` retiene una lectura por cada intento elegible; un intervalo positivo reduce pero no elimina ese coste y puede dejar una petición posterior sin guía fresca de zona de navegador.
