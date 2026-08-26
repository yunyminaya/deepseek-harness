# @deepseek-ai/dsh-repeat-tool-reminder

[English](README.md) | Español

Un rompedor de bucles de carácter consultivo, no una herramienta expuesta al modelo: nunca aparece en la lista de herramientas, nunca veta ni reescribe una llamada, y añade exactamente un comportamiento — observa el flujo de llamadas de herramienta de cada agent, cuenta las rachas de llamadas consecutivas a la misma herramienta con argumentos canónicos idénticos y, en las longitudes de racha configuradas, inyecta un recordatorio consultivo creciente que le dice al modelo que deje de repetirse, que vuelva a leer el último resultado y que cambie de enfoque o concluya. La decisión (reintentar de otra forma, recopilar más evidencia o terminar) queda por completo en manos del modelo: una llamada legítimamente repetida no se retrasa ni se bloquea por nada. Registro de decisión: [el Agent Note repeat-tool-reminder](../../../.agents/notes/archived/feature/2026-07-08-repeat-tool-guard.md).

## Config

```yaml
- id: repeat-tool-reminder
  name: '@deepseek-ai/dsh-repeat-tool-reminder'
  config:
    thresholds: [3, 5, 8]        # default; consecutive counts that trigger a reminder
    include: []                  # tool-name patterns to track; empty ⇒ all tools
    exclude: [todo_write]        # tool-name patterns transparent to the chain
    argumentsPreviewChars: 500   # default; cap on arguments quoted in the detailed reminder
```

`thresholds` falla ruidosamente al cargar el plugin: una lista vacía, un valor no entero, un valor inferior a 2 o un duplicado lanzan una excepción, nunca una vuelta silenciosa a los valores por defecto; `argumentsPreviewChars` rechaza igualmente cualquier cosa que no sea un entero >= 1. La lista se normaliza en orden ascendente; el PRIMER umbral entrega un empujón genérico corto, y cada umbral posterior entrega la forma detallada que nombra la herramienta, la longitud de la racha y los argumentos canónicos — truncados por la cabeza en `argumentsPreviewChars` con un marcador de recuento omitido, de modo que una carga útil de `write`/`edit` en bucle no pueda colarse sin límite en la siguiente petición (la clave de la cadena siempre compara la cadena canónica COMPLETA; el tope acota el recordatorio, nunca la detección).

Las entradas de `include`/`exclude` admiten comodines `*` y son predicados sobre las herramientas que existan en el momento de la llamada, no referencias a entradas del registro — un patrón que no coincida con ninguna herramienta registrada en ese momento NO es un error (`exclude: [mcp_*]` sigue siendo válido en un despliegue que no carga herramientas MCP), a diferencia de la comprobación de referentes de `toolOrder`.

## Semántica de la cadena

La clave de la cadena es `(tool name, canonical arguments)` — la canonicalización es una ordenación profunda de claves más `JSON.stringify`, así que los objetos de argumentos que solo difieren en el orden de las propiedades cuentan como idénticos. Una llamada idéntica a la llamada rastreada anterior incrementa el contador de consecutivas del agent; una llamada rastreada distinta lo reinicia a 1.

- **Las llamadas no rastreadas son transparentes para la cadena.** Una llamada excluida por `include`/`exclude` ni incrementa ni reinicia el contador, así que `grep X → todo_write → grep X` sigue contando como dos `grep X` consecutivas cuando `todo_write` está excluida. Esto es lo que hace útil la exclusión: las herramientas de anotación intercaladas en un bucle no deben blanquearlo.
- **Las llamadas denegadas cuentan.** La detección se asienta en `tools/post-execute`, que también se ejecuta para las llamadas que un listener de `tools/pre-execute` denegó — un modelo machacando una llamada denegada es exactamente el bucle que merece la pena romper.
- **Las llamadas sin agent se ignoran.** Un llamador directo de `ctx.tools.execute()` no tiene modelo al que recordar ni objeto agent vivo sobre el que fijar la clave.
- **Clave por agent.** El registro de herramientas tiene ámbito de contexto y los subagentes se intercalan por el mismo waterfall, así que un `WeakMap<Agent, Chain>` fija la clave de cada cadena por el objeto agent vivo; la repetición de un agent nunca dispara el recordatorio de otro. Un prompt de usuario (`agent/pre-step`) reinicia la cadena del agent que lo envía, y la vida del objeto acota la entrada débil sin un listener de eliminación.
- **Solo en memoria.** Una sesión reanudada desde persistencia arranca con una cadena nueva — el guard es un empujón heurístico, no un invariante registrado; los recordatorios posteriores son el coste aceptado.

## Entrega del recordatorio

Los recordatorios viajan en los `additionalContexts` de la decisión post-ejecución (fuente `{kind: 'plugin', plugin: 'repeat-tool-reminder'}`), nunca como reemplazo de `content`: el evento `tool/result` conserva la salida propia de la herramienta para auditoría. El bucle almacena el contexto en buffer y lo añade como `user/message` inyectado después de los resultados de herramienta del paso, que la sesión renderiza como un mensaje de usuario sintético normal — así el recordatorio es visible para el modelo, atribuible a su fuente y reconstruible desde el registro de la sesión sin ningún evento de sesión nuevo. El guard siempre delega mediante `next()` y antepone su recordatorio al array de contexto de la decisión aguas abajo (ambas variantes — una llamada bloqueada recibe igualmente el empujón); cada entrada conserva su propia fuente y metadatos.

## Model Experience

### Mensaje de contexto del primer umbral

#### Lo que ve el modelo

En el primer umbral configurado de repetición consecutiva, ese agent recibe el recordatorio siguiente. No se añade ningún schema de herramienta ni texto de llamada normal.

##### Recordatorio del primer umbral

```markdown
You are repeating the exact same tool call with identical arguments. Carefully analyze the previous result before calling again: if the task is not complete, try a different approach or different arguments instead of repeating the call.
```

#### Efecto de tokens

Cero tokens antes del umbral. El recordatorio es historial retenido para ese agent.

#### Efecto de KV Cache

Solo añadidura; el contenido recién visible sigue el prefijo de petición reutilizable y no invalida las entradas de caché KV existentes.

### Mensaje de contexto de umbrales posteriores

#### Lo que ve el modelo

Un umbral posterior recibe la plantilla de recordatorio detallado siguiente. Una vista previa de argumentos acotada termina exactamente en `… (+<omitted> more chars)`.

##### Recordatorio de umbrales posteriores

```markdown
Repeated tool call detected:
- tool: <toolName>
- consecutive_calls: <count>
- arguments: <canonicalArguments>
The repeated calls are not making progress. Do not call this tool with these exact arguments again. Inspect the latest result and choose a different action, different arguments, or finish the task if enough evidence has been gathered.
```

#### Efecto de tokens

Cada recordatorio es historial retenido; `argumentsPreviewChars` acota su texto de argumentos dependiente de los datos, mientras que los agents mantienen contadores independientes.

#### Efecto de KV Cache

Solo añadidura; el contenido recién visible sigue el prefijo de petición reutilizable y no invalida las entradas de caché KV existentes.

## Limitaciones conocidas y trabajo diferido

- **Detección solo de coincidencia exacta** — la canonicalización es una ordenación profunda de claves, así que las variantes casi idénticas (una ruta retocada, un espacio extra dentro de un valor) se escapan de la cadena; el emparejamiento difuso se rechaza a falta de evidencia de necesidad.
- **La compactación no reinicia las cadenas** — una cadena que atraviesa un checkpoint de compactación sigue contando.
- **Solo consultivo** — la escalada a `block` en un umbral alto no está implementada, aunque `PostToolDecision` ya soporta el bloqueo.
- **Sin compartición de cadenas entre subagentes** — las cadenas permanecen aisladas por agent; un padre y su subagente que repiten la misma llamada nunca se combinan.
- **El sondeo idempotente legítimo sigue recibiendo empujones** pasados los umbrales — las válvulas de presión son la config `thresholds`/`exclude`.
- **Pasado el umbral más alto, una cadena queda en silencio** — los recordatorios solo se disparan en los recuentos configurados exactos, nunca más allá de ellos.
