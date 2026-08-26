# @deepseek-ai/dsh-tool-web

[English](README.md) | Español

La suite de herramientas web orientadas al modelo — `web_search` y `web_fetch` — sobre el [seam de capacidad web](../web/README.md.es.md) (`ctx.web`). Es dueña únicamente de las preocupaciones orientadas al modelo: nombres de herramientas, JSON Schema, nombres de argumentos en snake_case, secciones del prompt, límite de recuento de resultados, formato de resultados, presentación HTML→markdown y la proyección de presentación de la UI — `presentCall`, `presentResult` (una tarjeta de resultado `card: 'web'` discriminada por `kind: 'search' | 'fetch'`) y el `output.presentationMeta` que transporta las fuentes de búsqueda estructuradas o el resumen de la obtención que el texto renderizado con pérdida no puede (véase la [Agent Note de la tarjeta de resultado web](../../../.agents/notes/implemented/feature/2026-07-30-web-result-card.es.md)). Todo el acceso web pasa por `ctx.web`; este paquete nunca importa un provider concreto. Ninguna herramienta expone un tiempo de espera orientado al modelo — el presupuesto cooperativo de tiempo de espera de llamada de herramienta de cada una se declara aquí mediante config (`fetchTimeoutMs`/`searchTimeoutMs`, adjuntado como `ToolDefinition.timeoutMs`) y lo aplica [`@deepseek-ai/dsh-tool-call-timeout-policy`](../../guard/timeout-policy/README.es.md) (un envoltorio de `tools/execute`). Las operaciones individuales reenvían `exec.signal`; una búsqueda de múltiples consultas lo fusiona con la cancelación del lote, de modo que una consulta fallida aborta a sus hermanas.

Cada herramienta se registra de forma independiente; un producto que solo quiera una desactiva la otra mediante config (`{ search: false }` / `{ fetch: false }`). La guía de búsqueda menciona `web_fetch` solo cuando fetch también está habilitado en la config; una composición solo de búsqueda en su lugar le dice al modelo que use los fragmentos devueltos y cite sus URLs.

## Herramientas

| Herramienta | Argumentos | Comportamiento |
|---|---|---|
| `web_search` | `queries` (string[] obligatorio) | Descubrimiento. Devuelve una respuesta opcional más las URLs de las fuentes. Ejecuta de una a `searchMaxQueries` búsquedas distintas de forma concurrente y fusiona sus fuentes en orden round-robin antes de aplicar el tope combinado de `searchMaxResults`. Un array de un solo elemento realiza una búsqueda. Las consultas duplicadas exactas se ejecutan una vez. Cualquier búsqueda fallida aborta el resto del lote, que se asienta antes de que la llamada devuelva un error. Ninguno de los dos límites está orientado al modelo. |
| `web_fetch` | `url` (string) | Recupera una URL concreta. Los cuerpos HTML se renderizan a markdown (turndown con tablas/tachado GFM); los cuerpos de texto pasan tal cual. Un estado no 2xx se notifica, no es un error. El tiempo de espera de la llamada de herramienta es política de despliegue (`dsh-tool-call-timeout-policy`), no un argumento del modelo. |

Ambas herramientas optan por la programación concurrente porque las lecturas del provider devuelven contenido sin mutar el estado del agent padre.

Los resultados de servicio normalizados son también los valores canónicos de las herramientas: `WebSearchResult` y `WebFetchResult`. Los renderizadores nativos conservan la respuesta/las fuentes y el texto del cuerpo obtenido; los topes de búsqueda/cuerpo del provider siguen siendo límites de adquisición, no una truncación solo de presentación.

## Config

| Clave | Predeterminado | Significado |
|---|---|---|
| `search` | `true` | Registra `web_search`. |
| `fetch` | `true` | Registra `web_fetch`. |
| `searchMaxResults` | `8` | Límite superior de fuentes devueltas por una llamada `web_search` (el seam trunca la lista de cada provider; la herramienta también limita una lista combinada de múltiples consultas). |
| `searchMaxQueries` | `4` | Límite superior de consultas aceptadas por una llamada `web_search`. El valor configurado aparece en su guía de prompt y en las descripciones del schema. |
| `fetchTimeoutMs` | `30000` | Presupuesto cooperativo de tiempo de espera de llamada de herramienta (ms) para `web_fetch`. |
| `searchTimeoutMs` | `30000` | Presupuesto cooperativo de tiempo de espera de llamada de herramienta (ms) para `web_search`. |
| `fetchMaxOutputChars` | `200000` | Tope de caracteres de fuente convertidos de forma síncrona y de una salida completa de `web_fetch` (encabezado, cuerpo renderizado y pie); un cuerpo recortado recibe el aviso de truncación cuando cabe. |

`searchMaxQueries` limita el array aceptado antes de la deduplicación exacta de strings, la expansión al provider y el crecimiento de la respuesta combinada del provider; la validación rechaza un array sobredimensionado antes de que comience cualquier búsqueda, y después el despacho conserva la primera aparición de cada consulta. Junto con los controles propios de cada provider, como `maxUses`, estos ajustes independientes son los presupuestos de búsqueda del producto; el seam genérico no expone la contabilidad de búsquedas nativas interna del provider. `fetchTimeoutMs`/`searchTimeoutMs` declaran el presupuesto cooperativo de tiempo de espera de cada herramienta (adjuntado como `ToolDefinition.timeoutMs`), aplicado por [`@deepseek-ai/dsh-tool-call-timeout-policy`](../../guard/timeout-policy/README.es.md); el schema orientado al modelo no expone ningún argumento de tiempo de espera. `fetchMaxOutputChars` limita tanto el trabajo de conversión síncrono como el resultado renderizado completo: solo se convierten esa cantidad de caracteres de fuente, y el encabezado, el prefijo convertido y el aviso de truncación se limitan luego juntos. El predeterminado deja margen por encima del tope de 100,000 caracteres de cuerpo del provider local, pero la expansión renderizada puede hacer que el límite final trunque igualmente el resultado.

```yaml
- id: tool-web
  name: '@deepseek-ai/dsh-tool-web'
```

## Registro estable

El registro de herramientas sigue la **habilitación** del producto, no la disponibilidad del backend. Una herramienta permanece visible incluso cuando su provider seleccionado falta, está mal configurado, es ambiguo o está temporalmente no disponible; el seam resuelve el provider en tiempo de ejecución y la ejecución falla con un `WebError` estructurado (p. ej. `WEB_PROVIDER_UNAVAILABLE`, `WEB_PROVIDER_AMBIGUOUS`), que `ToolRuntime.execute()` convierte en un resultado de herramienta de error que el modelo puede leer y sobre el que los hooks/UI pueden enrutar. Esto mantiene estable el schema del modelo sin hacer que el orden de carga de plugins, el estado de las credenciales o el timing de HMR formen parte del contrato orientado al modelo. Para eliminar por completo una herramienta web, desactívala aquí en la config.

La herramienta nunca llama a `available()` de un provider y nunca enumera providers — su única ruta de ejecución es `ctx.web.search()` / `ctx.web.fetch()`, y la indisponibilidad del provider le llega como los códigos `WebError` estructurados que la selección lanza en tiempo de ejecución. La selección del provider permanece íntegramente dentro del seam, con un único dueño.

## Experiencia de modelo

### System prompt

#### Lo que ve el modelo

Search y fetch aportan las guías de búsqueda web y de obtención web de abajo. Search elige su texto habilitado-para-fetch o solo-búsqueda desde la config en el momento del registro. Una restricción de herramienta con ámbito no elimina estas secciones registradas de forma independiente.

##### Guía de búsqueda web con fetch habilitado

```markdown
Use the web_search tool to discover current information on the web. The required queries array accepts 1–4 non-empty search queries; use a one-item array for a single search. It returns an optional answer plus a list of source URLs. Follow up with web_fetch when you need the full content of a specific result, and cite the relevant URLs as markdown links.
```

##### Guía solo de búsqueda web

```markdown
Use the web_search tool to discover current information on the web. The required queries array accepts 1–4 non-empty search queries; use a one-item array for a single search. It returns an optional answer plus a list of source URLs. Use the returned source snippets when available, and cite the relevant URLs as markdown links.
```

##### Guía de obtención web

```markdown
Use the web_fetch tool to retrieve the content of a specific HTTP(S) URL (for example a result from web_search). It returns the page content decoded to text. Cite the URL as a markdown link when you use its content.
```

#### Efecto en tokens

Coste fijo de guía por solicitud para cada herramienta habilitada en la config, incluso cuando una restricción oculta su schema. Alternar fetch o cambiar `searchMaxQueries` cambia la guía de búsqueda; alternar fetch también registra o elimina la sección de fetch.

#### Efecto en la KV cache

Prefijo estable mientras las herramientas habilitadas, el ámbito y el texto de la guía no cambien. La habilitación en config — incluido alternar la rama de guía de búsqueda de fetch —, cambiar `searchMaxQueries` o el ciclo de vida del plugin pueden invalidar la reutilización desde la primera sección de prompt cambiada; las restricciones de schema con ámbito no la eliminan.

### Schemas de herramientas

#### Lo que ve el modelo

El modelo ve los [schemas `web_search` y `web_fetch` generados](../../../docs/tool-catalog.es.md#deepseek-aidsh-tool-web). Los presupuestos de recuento de resultados y de tiempo de espera son ajustes de despliegue, no argumentos del modelo.

#### Efecto en tokens

Coste fijo de schema por solicitud para un `searchMaxQueries` resuelto; la desactivación en config elimina tanto el schema como la guía, mientras que una restricción con ámbito elimina solo el schema.

#### Efecto en la KV cache

Prefijo estable mientras las definiciones, el tope de consultas resuelto y la visibilidad no cambien. La habilitación en config, cambiar `searchMaxQueries`, el ciclo de vida del plugin o las restricciones con ámbito pueden invalidar la reutilización desde el primer token de schema cambiado.

### Resultado de búsqueda

#### Lo que ve el modelo

A la respuesta opcional propiedad del provider le siguen `Sources:` y líneas dependientes de los datos con la forma exacta `- [<title-or-url>](<url>)`, opcionalmente con el sufijo ` — <snippet> (<publishedAt>)`. Una llamada de múltiples consultas ejecuta cada string de consulta exacto una vez, conservando su primera posición; etiqueta la respuesta de cada provider con la consulta de origen como encabezado markdown, deduplica las fuentes por URL y toma una fuente de cada consulta en cada rango antes de avanzar al siguiente. Sin respuesta ni fuentes, el resultado dice `No results found.` Una lista limitada añade `(Showing the first <count> sources. Refine the query for more.)`; todo resultado termina con `Cite the relevant URLs above as markdown links in your answer.`

#### Efecto en tokens

Los resultados dependientes de los datos se reenvían hasta la compactación; la expansión de consultas está limitada por `searchMaxQueries` y las fuentes por `searchMaxResults`.

#### Efecto en la KV cache

Solo append; el contenido recién visible sigue al prefijo de solicitud reutilizable y no invalida las entradas de KV cache existentes.

### Fallo de búsqueda

#### Lo que ve el modelo

Si falla cualquier consulta de una llamada de múltiples consultas, `web_search` aborta las demás búsquedas, espera a que cada búsqueda iniciada se asiente, descarta los resultados correctos y devuelve `Error: <message>` para el primer fallo.

#### Efecto en tokens

Solo el resultado de error retenido añade tokens; los resultados correctos descartados no entran en el historial del modelo.

#### Efecto en la KV cache

Solo append; el error sigue al prefijo de solicitud reutilizable y no invalida las entradas de KV cache existentes.

### Resultado de obtención

#### Lo que ve el modelo

Una obtención correcta es exactamente `Fetched <finalUrl> (HTTP <statusCode>)`, una línea en blanco y el cuerpo decodificado propiedad del provider. La truncación añade una línea en blanco y `(Content truncated. Fetch a more specific URL or section for the full text.)`; los fallos se convierten en `Error: <message>`. Las consultas y las URLs permanecen en el historial de llamadas.

#### Efecto en tokens

Los topes del provider limitan el tamaño del cuerpo; los argumentos y resultados de llamada retenidos se reenvían hasta la compactación, y la política de tiempo de espera puede sustituir un resultado tardío por un error corto.

#### Efecto en la KV cache

Solo append; el contenido recién visible sigue al prefijo de solicitud reutilizable y no invalida las entradas de KV cache existentes.

### Errores de argumentos

#### Lo que ve el modelo

La validación del schema rechaza antes de la ejecución un campo `queries` ausente o no-array y elementos de array que no sean strings. Los errores de valor se convierten exactamente en `Error: queries must contain at least one query`, `Error: queries must contain at most 1 query` cuando el tope configurado es uno, `Error: queries must contain at most <count> queries` para topes mayores, `Error: each query must be a non-empty string` o `Error: url must be a non-empty string`.

#### Efecto en tokens

Solo la llamada fallida añade estos tokens retenidos.

#### Efecto en la KV cache

Solo append; el contenido recién visible sigue al prefijo de solicitud reutilizable y no invalida las entradas de KV cache existentes.

## Limitaciones conocidas y trabajo pendiente

- **No existe un contador de búsquedas nativas de todo el lote** — `searchMaxQueries` limita las llamadas a `ctx.web.search`, pero un provider puede realizar varias búsquedas nativas dentro de cada llamada. Por ejemplo, un provider respaldado por modelo configurado con `maxUses` puede permitir hasta `searchMaxQueries × maxUses` búsquedas nativas; `searchMaxResults` limita solo las fuentes combinadas devueltas al llamador. Los despliegues controlan el coste mediante estos ajustes independientes de consumidor y de provider porque el seam genérico no conoce las unidades de búsqueda internas del provider.
- **La conversión HTML→markdown se degrada con entradas que GFM no puede representar con seguridad** — [turndown](https://github.com/mixmark-io/turndown) (con tablas/tachado GFM) convierte como máximo `fetchMaxOutputChars` caracteres de fuente a través de un DOM real. Un guardia léxico conservador de 512 niveles deja pasar los cuerpos profundamente o ambiguamente anidados como HTML bruto, las excepciones de conversión hacen lo mismo, y se ignora el `colspan` de tablas porque GFM no tiene representación de celdas que abarquen; estos límites evitan bloquear el event loop o expandir la salida a partir de un atributo numérico no confiable ([decisión de dependencia archivada](../../../.agents/notes/archived/simplification/2026-07-26-turndown-for-tool-web-html-markdown.md)).
- **La API orientada al modelo es mínima por diseño, con promociones aplazadas** — `max_results` sigue siendo un límite de config (no un argumento del modelo), y `web_fetch` solo acepta `url` (sin modo `format`/`prompt`/resumen LLM); ambos son pasos posteriores nombrados en [la Agent Note del seam](../../../.agents/notes/implemented/architecture/2026-06-24-web-capability-seam.es.md).
- **No hay política de permisos específica de web** — ambas herramientas se ejecutan sin solicitar `ctx.approval`; un despliegue que necesite confirmación debe añadir una política de `tools/pre-execute`, y el paquete no define concesiones persistentes de URL/dominio.
