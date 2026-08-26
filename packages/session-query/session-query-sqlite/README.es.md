# @deepseek-ai/dsh-session-query-sqlite

[English](README.md) | Español

Provider concreto de `ctx.sessionQuery`. `SqliteSessionQueryEngine` hereda las lecturas exactas, las trazas y los filtros independientes del provider del paquete de Service Definition e implementa sus dos métodos de texto completo con SQLite FTS5. La búsqueda usa el corpus lógico de sesiones con preferencia por lo vivo y agrupa los resultados entre sesiones por su evento más fuerte.

## Contrato de búsqueda

`searchSessions(request, exec?)` devuelve páginas de `SessionSearchHit` en todo el corpus; `searchEvents(request, exec?)` devuelve páginas de `SessionEventSearchHit` dentro de una sesión. Las consultas son frases literales obligatorias, recortadas y normalizadas de espacios en blanco. La sintaxis de FTS5 como las comillas, `OR`, `NEAR` y `*` se trata como datos, no como sintaxis MATCH ejecutable. Los filtros de metadatos son predicados SQL parametrizados que se aplican antes del ranking. Para mantener el MATCH de SQLite FTS5 en un contexto de predicado externo soportado, las peticiones entre sesiones pueden compilar como mucho 14 predicados de filtro combinados de sesión y evento; las peticiones dentro de una sesión pueden compilar como mucho 13 predicados de filtro porque el predicado fijo de la sesión objetivo consume una ranura. Cada extremo de rango se compila como un predicado. Una petición que supere cualquiera de los dos presupuestos de predicados o el límite portable de SQLite de 32.766 bindings totales, incluidos los valores fijos de consulta y paginación, falla con `SESSION_QUERY_INVALID_FILTER` antes de la preparación de la sentencia.

La relevancia es comparable entre orígenes en las tablas persistentes y TEMP: recuento descendente de tramos de coincidencia resaltados reales de FTS5 y luego longitud ascendente de code points del documento almacenado. El tiempo del evento, el id de sesión cuando proceda y seq rompen los empates restantes. Los resultados entre sesiones exponen el evento seleccionado como `bestMatch`; ambos ámbitos derivan texto plano normalizado de espacios en blanco de las posiciones de resaltado de FTS5 y lo acotan en code points de Unicode. Los cursores son valores branded opacos, se vinculan a la petición normalizada y a la instancia de servicio, y fallan cuando cambia la generación pertinente. Un cursor dentro de una sesión sobrevive a cambios de sesiones no relacionadas; un cursor entre sesiones no.

Las tres superficies (`current`, `shadowed` y `log-only`) son buscables por defecto. Pasa un filtro de superficie para reducirlas.

## Ciclo de vida del origen y del índice

El servicio exige `ctx.sessions` y observa el `ctx.sessionPersistence` opcional de forma dinámica. Una máquina de estados serializada compara las revisiones de instantáneas durables ligeras cualificadas por origen, inspecciona sin mutar solo los logs nuevos o cambiados, extrae los documentos semánticos compartidos, concilia los cambios transaccionalmente y ejecuta la consulta. Las consultas de sesión nunca invocan el `load()` reparador de crashes del backend de persistencia; un dueño que se adjunte durante la inspección no puede mutar su log, y el reintento de observación estable hace que el resultado prefiera lo vivo. La fila viva TEMP sigue registrando la disponibilidad persistida, y la base durable se refresca después de que ese dueño vivo se desprenda. Las consultas repetidas y la reapertura sin cambios del mismo store no hacen ninguna inspección completa del log durable; cambiar de store, u observar orígenes nuevos, cambiados, borrados o reparados externamente por load, concilia en la siguiente observación estable. Un fallo de origen o de transacción no confirma nada, y la siguiente búsqueda reintenta.

`openAt: startup` es el default: la activación del servicio importa `node:sqlite`, abre el handle y falla antes de la publicación cuando el índice es inválido. `openAt: first-search` publica el servicio como ACTIVE sin importar el módulo de SQLite ni abrir un handle; las primeras búsquedas concurrentes comparten una promesa de disponibilidad, y la disposición antes de cualquier búsqueda no abre nada. Este modo soporta composiciones que necesitan una salida limpia de arranque de Node 22 al diferir la advertencia experimental de SQLite hasta la primera búsqueda real; no suprime una advertencia en ese punto. Una base de datos inválida falla igualmente la primera búsqueda en lugar de la activación del servicio. `openAt: never` apaga la búsqueda de texto completo para el despliegue: `searchSessions` y `searchEvents` fallan con `SESSION_QUERY_SEARCH_DISABLED` antes de cualquier normalización de petición, node:sqlite nunca se importa ni se abre, y no corre ninguna observación ni conciliación de orígenes, mientras que cada lectura, filtro y traza exactos heredados en `ctx.sessionQuery` siguen funcionando.

Las filas FTS persistidas viven en una base de datos derivada dedicada. Las tablas TEMP locales a la conexión contienen las filas vivas, que hacen sombra a la base durable para la misma sesión y la revelan cuando el dueño vivo desaparece. Desmontar la persistencia oculta las filas durables sin descartar la caché; volver a montarla la concilia. Cerrar o reabrir la base de datos elimina cada overlay vivo conservando las filas persistidas.

La base de datos es desechable pero el reset está protegido: cada versión de schema reconocida rechaza tablas de usuario desconocidas antes de mutar el modo journal, y solo un schema incompatible reconocido que contenga tablas derivadas se reconstruye en su sitio. Se rechaza una base de datos ajena o canónica. Nunca apuntes `path` a la base de datos de persistencia de sesiones. En filesystems con modos POSIX, los directorios y las bases de datos ausentes se crean solo para el dueño (`0700` y `0600` antes del umask del proceso), y los sidecars de SQLite heredan el modo de la base de datos; los modos existentes se conservan. Exactamente un servicio en un proceso es el dueño de una ruta de índice derivado; los escritores externos o un segundo proceso no están soportados porque las generaciones y el estado de sombra TEMP son propiedad de la conexión.

## Configuración

| Clave | Default | Contrato |
|---|---:|---|
| `path` | obligatorio | Ruta SQLite dedicada del índice derivado; se soporta `:memory:`. Las rutas de filesystem ausentes se crean solo para el dueño en filesystems POSIX. |
| `openAt` | `startup` | `startup` abre antes de que se complete la activación del servicio; `first-search` difiere el módulo y el handle de SQLite hasta la búsqueda; `never` desactiva la búsqueda de texto completo (fallos tipados `SESSION_QUERY_SEARCH_DISABLED`) mientras las lecturas heredadas siguen disponibles. |
| `journalMode` | `wal` | `wal`, `delete`, `truncate` o `persist`. |
| `defaultLimit` | `20` | Tamaño de página cuando una petición omite `limit`; como mucho `Number.MAX_SAFE_INTEGER - 1`. |
| `maxLimit` | `100` | Mayor tamaño de página de petición aceptado; como mucho `Number.MAX_SAFE_INTEGER - 1`. |
| `snippetChars` | `240` | Longitud máxima de snippet en code points de Unicode. |
| `readWindowMax` | `50` | Recuento máximo de eventos crudos `before` o `after` para el `readEvent()` heredado. |
| `persistedInspectConcurrency` | `4` | Inspecciones concurrentes máximas de logs persistidos para las lecturas de lote heredadas; debe ser un entero seguro positivo. |

## Tokenizador y límites

El índice usa FTS5 `unicode61`. La compensación es el recall por token/frase en lugar del recall arbitrario de subcadenas: `AI` no coincide con el token `BRAID`. Usa `ctx.sessionQuery.filterEvents()` con una cláusula `text` cuando se necesite un escaneo literal de subcadena flexible con espacios. El NUL se rechaza en las consultas; los marcadores de resaltado reservados y el NUL en los documentos se normalizan antes de indexar para que los marcadores de presentación no puedan chocar con el texto de origen.

Las señales de abort detienen el trabajo encolado y fluyen sin cambios por el listado de instantáneas y la inspección sin mutación. Una vez que empieza el trabajo de origen, la máquina de estados serializada espera ella misma esa promesa del backend —aunque un backend ignore la cancelación— y luego comprueba la señal antes de empezar cualquier trabajo posterior de listado, inspección, conciliación o consulta. El caller observa por tanto la cancelación solo después de que el trabajo de backend iniciado quede en reposo, y una búsqueda posterior no puede entrar en el serializador mientras esa limpieza esté pendiente. La API síncrona `DatabaseSync` de Node no puede interrumpir una sentencia de metadatos o MATCH que ya se esté ejecutando en el hilo de JavaScript; las señales se comprueban inmediatamente antes y después de esas llamadas no apropiativas.

## Experiencia del modelo

Ninguna, ya que este backend de búsqueda de confianza devuelve resultados solo a los callers y no registra ningún prompt, schema, herramienta ni mensaje orientado al modelo.

#### Efecto en la caché KV

Ninguno; este paquete ni ensambla ni envía una petición de provider.

## Limitaciones conocidas y trabajo diferido

- **Sin autorización de caller** — es un servicio de confianza con alcance de contexto; una herramienta de modelo o una UI deben imponer su propia política de acceso.
- **Ejecución síncrona de consultas** — `DatabaseSync` bloquea el hilo de JavaScript durante la ejecución de MATCH y no puede interrumpir una sentencia ya en ejecución.
- **Recall por tokens, no subcadenas arbitrarias** — el tokenizador `unicode61` no coincide con subcadenas dentro de un token mayor; usa `filterEvents()` para escaneos literales.
- **Índice derivado de dueño único** — un servicio en un proceso debe ser el dueño de cada ruta de índice; los escritores externos y el uso compartido multiproceso no están soportados.
