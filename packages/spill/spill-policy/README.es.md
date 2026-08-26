# @deepseek-ai/dsh-spill-policy

[English](README.md) | Español

La **política de spill de resultados de herramientas**: un transformador de `tools/post-execute` que mantiene los resultados de herramientas de texto plano sobredimensionados fuera del contexto del modelo. Cuando un resultado final supera `maxInlineBytes`, guarda el texto COMPLETO a través de [`ctx.spillStore`](../spill) y sustituye el resultado visible para el modelo por una vista previa acotada de cabeza/cola más el localizador del backend y su pista de recuperación.

Este plugin **no registra ningún servicio** y no posee mecanismos de almacenamiento ni de vista previa: la vista previa es de [`@deepseek-ai/dsh-output-retention`](../../util/output-retention) (`TextRetainer`), y el almacenamiento es `ctx.spillStore`. Solo decide CUÁNDO hacer spill y compone el aviso.

## Configuración

| Clave | Valor por defecto | Significado |
|---|---|---|
| `maxInlineBytes` | *(omitido)* | Límite de contexto visible para el modelo para un resultado de texto plano, en bytes UTF-8 (un entero no negativo; validado en la carga). **Omitirlo desactiva la política por completo** (el plugin no registra nada). Cuando se fija, un resultado mayor se hace spill y se sustituye por una vista previa derivada del mismo presupuesto (división cabeza/cola). |

## Comportamiento

1. Deja correr la herramienta (delega vía `next()`, de modo que acota lo que un hook posterior haya aceptado).
2. Omite las ejecuciones anidadas (`exec.parent` está presente — su copia DURADERA queda acotada por el brazo del dispatch-log de abajo), las sustituciones de valores aceptados (el registro debe revalidarlas y rerenderizarlas), `read` (evita un bucle `read → spill → read de nuevo`) y cualquier decisión que no sea `accept` (el feedback correctivo de un `block` pasa de largo).
3. Aplana el contenido aceptado solo cuando es **texto plano** (todos los bloques `text`); un resultado con cualquier bloque que no sea texto se deja intacto.
4. Si su tamaño UTF-8 es `≤ maxInlineBytes`, lo deja sin cambios.
5. En caso contrario, guarda el texto completo y sustituye el resultado por una vista previa + este aviso, dimensionados para que la sustitución completa (vista previa + línea en blanco + aviso) quepa en `maxInlineBytes` — el coste en bytes del aviso se reserva del presupuesto, así que la vista previa se encoge para caber y el resultado visible para el modelo nunca supera el límite:

   ```text
   <retained head/tail preview>

   (Omitted N bytes. Full formatted result stored at: /…/session-…/…-web_fetch.txt. Use read with offset/limit, or grep this path to search within it.)
   ```

   Cuando solo el aviso llena el presupuesto (un límite diminuto o un localizador largo), la vista previa queda vacía y solo se devuelve el aviso. Si incluso esa sustitución de solo aviso excediera `maxInlineBytes`, la política conserva el resultado inline — nunca emite una sustitución por encima del límite (y una sustitución dentro del límite siempre es más pequeña que el original, así que esto también significa que hacer spill nunca añade bytes).

**Best-effort (de mejor esfuerzo):** sin dueño de sesión, sin backend de `ctx.spillStore` o con un rechazo de `saveText` ⇒ la política registra una advertencia y devuelve el resultado original. Un fallo de spill nunca convierte una llamada exitosa en un `isError` ni oculta el resultado inline. Una sustitución exitosa cambia solo `content`; el valor canónico programático se conserva.

**El brazo del dispatch-log:** un segundo listener en `tools/code-dispatch-log` aplica el mismo límite, la misma tubería de sustitución y los mismos fallbacks best-effort a la copia DURADERA del resultado de cada sub-llamada de `run_code` (etiqueta de artefacto `dispatch`, con clave por el id de la sub-llamada). El valor del programa no se toca — ya cruzó la frontera del worker completo — y las sub-llamadas de `read` también quedan acotadas: una copia del log no es contexto del modelo, así que el bucle de volver a leer no puede ocurrir, y `read` es precisamente la herramienta que produce logs enormes ([fundamento](../../../.agents/notes/implemented/feature/2026-07-26-code-dispatch-log-spill.es.md)).

## Alcance

La política ve solo el resultado FINAL formateado visible para el modelo — no el recurso interno ni el valor canónico de una herramienta. Si un provider ya truncó (p. ej. `web-fetch-http.maxBodyChars`), el artefacto de spill contiene el resultado formateado completo que devolvió la herramienta, no la fuente original completa. Los límites de provider/recurso siguen siendo obligatorios e independientes. `glob`/`grep` son dueños del spill de presentación a nivel de elemento porque sus valores adquiridos completos siguen existiendo antes de renderizar; los streams de bash son dueños del spill en el momento de la adquisición. La política genérica antepone su listener waterfall (cascada de eventos) y luego delega, de modo que las proyecciones asíncronas ordinarias propiedad de la herramienta completen antes del acotado genérico de bytes, independientemente del orden de carga de los plugins. Consulta la [Agent Note de spill de salida de herramientas](../../../.agents/notes/implemented/architecture/2026-07-08-tool-output-spill-files.es.md).

## Experiencia del modelo

### Resultado de texto plano sobredimensionado

#### Qué ve el modelo

Los resultados en `maxInlineBytes` o por debajo, los resultados anidados, los resultados de `read`, las decisiones bloqueadas y los resultados que contienen bloques que no son texto quedan sin cambios. Un resultado de texto plano sobredimensionado visible para el modelo se convierte en una vista previa acotada de cabeza/cola seguida de `(Omitted <bytes> bytes. Full formatted result stored at: <locator>. <retrievalHint>)`; los fallos de almacenamiento o de propiedad dejan visible el resultado original.

#### Efecto de tokens

Una sustitución exitosa tiene como mucho `maxInlineBytes` bytes UTF-8 y permanece en el historial hasta la compactación; el texto completo del spill no se reenvía al modelo.

#### Efecto de KV Cache

Solo de añadido; el contenido recién visible sigue el prefijo de solicitud reutilizable y no invalida las entradas existentes de la caché KV.

## Limitaciones conocidas y trabajo diferido

- **Solo los resultados finales de texto plano se pueden hacer spill** — los resultados de contenido mixto, el feedback bloqueado y `read` pasan de largo; la truncación del provider o la retención propiedad de la herramienta que ocurrieron antes no se pueden recuperar aquí.
- **Un aviso que no cabe desactiva la sustitución para esa llamada** — un límite diminuto o un localizador largo dejan el original sobredimensionado inline después de que el backend ya haya guardado un spill sin referencias.
