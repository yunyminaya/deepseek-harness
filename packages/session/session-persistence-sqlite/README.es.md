# @deepseek-ai/dsh-session-persistence-sqlite

[English](README.md) | Español

Un provider de `SessionPersistence` SQLite opcional. Almacena las secuencias elegibles de `assistant/chunk` en filas físicas empaquetadas, comprime selectivamente con Zstandard los payloads grandes y codifica por diferencias las secuencias de procedencia mientras restaura el `SessionEvent[]` lógico exacto. Ninguna composición distribuida lo selecciona; los despliegues montan este paquete explícitamente y proporcionan la ruta de su base de datos.

`locate(meta)` devuelve `undefined` porque todas las sesiones comparten una única base de datos. El provider no expone ningún artefacto crudo por sesión.

## Modelo de almacenamiento

El schema 17 conserva las tablas ROWID ordinarias y el índice compuesto de clave primaria `events(session_id, seq)`. Las filas escalares almacenan un evento lógico. Las filas empaquetadas usan `text-chunks`, `reasoning-chunks` o `tool-call-chunks` como `type` físico; `seq` y `time` identifican el primer evento representado, y `data` contiene el payload compartido de fragmentos empaquetados. Las filas empaquetadas fijan `ignorable=0` como discriminador físico y dejan `source_event_seqs` y `surface_op` en `NULL`; las filas escalares usan `ignorable=1` solo para eventos lógicos ignorables y `NULL` en el resto. Un evento lógico ignorable futuro puede por tanto reutilizar un nombre de etiqueta de almacenamiento sin decodificarse como fila empaquetada. Estas etiquetas son registros de almacenamiento, no miembros de `SessionEventMap`.

El schema 17 es dueño de su códec localmente en lugar de importar la implementación mutable de otro formato de persistencia. Solo se empaquetan las formas delta exactas y consecutivas de texto, razonamiento o llamadas de herramienta del mismo bloque. Los campos desconocidos, los metadatos de surface, los huecos de secuencia, la identidad de bloque/llamada incompatible y las marcas de tiempo inseguras permanecen escalares. Una fila empaquetada representa como máximo 1.024 eventos y como máximo 1 MiB de `data` UTF-8 sin comprimir; las secuencias más largas se particionan sin cambiar los eventos lógicos. Las lecturas reconstruyen cada número de secuencia, marca de tiempo, frontera de tokens, fragmento de argumento y payload originales antes de devolver los datos al coordinador de persistencia.

El `data` serializado de menos de 4 KiB permanece como `TEXT` de SQLite. En o por encima de ese umbral, el escritor usa Zstandard nivel 3 y almacena un `BLOB` solo cuando el marco es más pequeño que el texto original; el lector lo descomprime antes de la validación UTF-8 y del análisis JSON. `source_event_seqs` sigue siendo el array ordenado completo de procedencia. Su primera secuencia es un varint sin signo y cada secuencia posterior es un delta con signo codificado con varints ZigZag, almacenado como `BLOB`; ninguna fuente se omite ni se convierte en un rango.

Cada anexión mantiene `BEGIN IMMEDIATE`, valida la cola física acotada, empaqueta solo el nuevo lote duradero, inserta esos registros e incrementa la revisión de sesión una vez. Las anexiones normales nunca borran ni reemplazan una fila de evento anterior. La ventana de write-behind por defecto de 200 ms comprime por tanto los streams de alta frecuencia mientras el volumen físico de escritura permanece proporcional a los lotes recién duraderos en lugar de reescribir repetidamente un valor empaquetado en crecimiento. Una comprobación de cola lógica a nivel de almacenamiento rechaza un escritor obsoleto antes de la mutación.

Las lecturas completas escanean las filas físicas en orden del primer seq lógico. Una pasada inversa encuentra el último `turn/end` válido sin retener copias decodificadas de cada fila física; la pasada directa decodifica y valida una fila física cada vez hacia el array de eventos lógicos devuelto. `readFrom(id, fromSeq)` examina los predecesores empaquetados solo dentro del tramo máximo de filas y ancla el sufijo en el más temprano que pueda contener `fromSeq`; esto incluye un rango de eventos que comienza dentro de una fila empaquetada, detecta corrupción física solapada y no analiza filas escalares anteriores no relacionadas. Una fila empaquetada malformada es todo o nada: la corrupción confirmada rechaza, mientras que una fila final rasgada se borra desde su base física durante la recuperación mutante. La reparación vuelve a leer la cola bajo el bloqueo de escritura y rechaza un marcador obsoleto antes de borrar nada. El `data` empaquetado que supera el límite de bytes del schema se rechaza antes del análisis JSON.

## Compatibilidad de schemas

Una base de datos virgen se inicializa directamente en el schema 17. Los schemas más antiguos, las identidades de aplicaciones ajenas, las bases de datos no vírgenes sin versión y los objetos de schema incompatibles se rechazan; este provider de pre-release no suministra migración. Cada sentencia y pragma fijo vive en un recurso `.sql` empaquetado; los valores usan parámetros de SQLite y el código de runtime nunca ensambla texto de consulta.

## Configuración (schemastery)

```ts
interface Config {
  path: string
  journalMode?: 'wal' | 'delete' | 'truncate' | 'persist'
  busyTimeoutMs?: number
  preparedSessionCacheSize?: number
  writeBatchMaxDelayMs?: number
}
```

`journalMode` tiene por defecto `wal`, `busyTimeoutMs` tiene por defecto `5,000`, `preparedSessionCacheSize` tiene por defecto `5` y `writeBatchMaxDelayMs` tiene por defecto `200`. El tiempo de espera acota cada espera síncrona de bloqueo de SQLite. Como SQLite puede devolver `SQLITE_BUSY` inmediatamente al cambiar el modo de journal, la apertura en frío cede entre intentos y no inicia ningún intento más después de un límite de reintentos relativo a la apertura. Una llamada síncrona a SQLite en curso puede terminar después de ese límite. El provider deshabilita los trusted schemas y la E/S con mapeo de memoria en cada conexión, y después lee de vuelta ambos ajustes. El modo de journal seleccionado también se lee de vuelta y debe coincidir; las bases de datos en memoria aceptan explícitamente el resultado `memory` de SQLite. Tras seleccionar el journal, el provider fija `synchronous=FULL` y lo verifica para que los valores por defecto de compilación de SQLite no puedan debilitar la durabilidad de la anexión confirmada. En POSIX, el directorio padre y el archivo de la base de datos deben pertenecer al usuario actual, el padre no debe ser escribible por grupo/mundo y el archivo no debe tener permisos de grupo/mundo. Los enlaces simbólicos y los archivos no regulares se rechazan. Windows también rechaza los enlaces simbólicos y los archivos no regulares, pero los despliegues siguen siendo responsables de restringir los ACL del directorio y del archivo al usuario del harness. Los fallos de ruta y de propiedad rechazan la inicialización del plugin. El SQLite de Node se carga de forma perezosa en la primera operación de persistencia; el import suprime solo el `ExperimentalWarning` exacto de SQLite de Node 22. Los fallos de identidad del store y de schema rechazan esa operación antes de exponer o mutar datos.

## Experiencia de modelo

### Historial de conversación reanudado

#### Lo que ve el modelo

Nada específico de SQLite. El resume restaura los mismos eventos lógicos y mensajes derivados que JSONL; las etiquetas físicas empaquetadas nunca llegan a prompts, herramientas, reproducción ni entrega en vivo de `session/event`.

#### Efecto en tokens

Cero tokens de solicitud en vivo. El resume paga solo por el historial lógico retenido y el envoltorio de solicitud actual.

#### Efecto en la caché KV

El empaquetado físico no muta los prefijos de solicitud. La reutilización de la caché del provider depende del historial reconstruido, del envoltorio actual y de la ruta del modelo exactamente igual que con otros backends de persistencia.

## Limitaciones conocidas y trabajo diferido

- **Diseño provisional específico de SQLite** — Esta implementación centrada en la eficiencia se basa en [morlay/session-persistence-rdb](https://github.com/morlay/session-persistence-rdb). Un diseño unificado de base de datos relacional con múltiples backends y schemas configurables queda diferido; ni la estabilidad del schema ni el soporte de migración están garantizados durante el desarrollo de pre-release.
- **El empaquetado sigue las fronteras de lote duradero** — las secuencias compatibles divididas por la ventana de write-behind o un flush explícito siguen siendo registros físicos separados; esto evita reescribir filas anteriores a costa de una proporción de empaquetado dependiente del tiempo.
- **Compresión síncrona** — las llamadas a SQLite y Zstandard de Node bloquean el hilo de JavaScript; el umbral de 4 KiB limita el trabajo por marco para los registros pequeños.
- **`DatabaseSync` bloquea el event loop** — la reducción de filas físicas no hace asíncronas las operaciones de SQLite.
- **Las esperas de busy bloquean el event loop** — SQLite espera dentro de las llamadas síncronas de `DatabaseSync`; solo una transición de modo de journal con busy cede entre intentos, y el límite relativo a la apertura impide otro intento en lugar de interrumpir una llamada activa.
- **Los lectores SQL externos deben entender las etiquetas físicas** — los Consumers soportados leen a través de este provider en lugar de tratar cada `events.type` como un tipo de evento lógico.
- **Sin borrado ni compactación histórica en segundo plano** — las anexiones normales son solo de inserción.
