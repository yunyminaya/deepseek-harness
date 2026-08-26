# @deepseek-ai/dsh-tool-terminal

[English](README.md) | Español

Seis herramientas orientadas al modelo sobre `ctx.terminals`: `terminal_open`, `terminal_send`, `terminal_read`, `terminal_signal`, `terminal_close` y `terminal_list`. Cada operación exige el `Agent` iniciador exacto, de modo que el modelo no puede dirigirse al terminal de otro agent aunque conozca su id.

`terminal_send(run_in_background: true)` reutiliza `ctx.jobs`; la comprobación previa del job y la reserva exclusiva de envío por sesión del servicio PTY ocurren antes de devolver el id del job, la finalización se recoge con `job_output`, y `job_kill` entrega `SIGINT` al grupo de procesos en primer plano. Los envíos en primer plano usan tarjetas de llamada/resultado de terminal. Los envíos en segundo plano usan una tarjeta de ejecución genérica; open, read, signal, close y list usan tarjetas genéricas `execute`, `read`, `execute`, `delete` y `read`, respectivamente. Ninguna declara ubicaciones de fuente.

## Configuración

| clave | por defecto | significado |
|---|---:|---|
| `enableRunInBackground` | `true` | exponer y aceptar `run_in_background`; false omite el campo del schema y rechaza un argumento no declarado forzado |
| `maxResultBytes` | `262144` | tope UTF-8 (mínimo `64`) para cada resultado completo de terminal o salida de job PTY tras espera, sesión, paginación, truncamiento y metadatos de estado de tarea |

Ambos valores se validan en la carga. El tope mínimo de resultado mantiene visible cada id de sesión o job emitido por el registro en su acuse de creación. Cuando un resultado supera `maxResultBytes`, el renderizado reserva espacio para los metadatos de control y un marcador de truncamiento cuando caben; los cortes conservan los límites UTF-8. El callback de contenido final de cada definición de terminal aplica el mismo tope tras fallos, denegaciones, cortocircuitos, reemplazos o bloqueos de política normalizados previos, alrededor y posteriores a la ejecución; un resultado de política estructurado en varios bloques conserva su forma.

## Model Experience

### Prompt del sistema

#### Lo que ve el modelo

El plugin contribuye esta sección de guía fija:

##### Terminal guidance

```markdown
Use a terminal session only when work needs persistent terminal state or interactive stdin; prefer shell/read/write/edit for bounded one-shot operations. Track every terminal session id and close sessions that no longer matter. An inferred_idle or timeout result does not prove the foreground command exited.
```

#### Efecto de tokens

Pequeño coste fijo de entrada en cada petición mientras el plugin está activo.

#### Efecto de caché KV

Estable respecto al prefijo mientras el ámbito de registro y el texto de guía no cambian.

### Schemas de herramientas

#### Lo que ve el modelo

Los seis schemas generados se listan en la [sección de catálogo de `dsh-tool-terminal`](../../../docs/tool-catalog.es.md#deepseek-aidsh-tool-terminal). Sus tokens de schema fijos están presentes siempre que este plugin está activo; el filtrado de herramientas por agent puede ocultarlos.

#### Efecto de tokens

Coste de schema fijo en las peticiones donde las herramientas son visibles.

#### Efecto de caché KV

Estable respecto al prefijo mientras la visibilidad y las definiciones de herramientas no cambian.

### Resultados de herramientas y contexto de tarea

#### Lo que ve el modelo

El spawn devuelve el id y un MOTD acotado. Send/read devuelven texto de terminal acotado más marcadores de preparación/historial. El modo en segundo plano devuelve un id de job genérico. Cada resultado de texto único propiedad del terminal o producido por política está limitado por `maxResultBytes` tras errores, denegaciones, cortocircuitos, reemplazos, bloqueos y texto de estado de job genérico normalizados de herramienta o pipeline. Los resultados de política estructurados en varios bloques conservan su forma. Los resultados permanecen en el historial de la sesión hasta la compactación; las lecturas incrementales de tarea no repiten la salida consumida. Quien llama programáticamente recibe instantáneas de sesión tipadas, DTOs acotados de lectura/envío del provider, resultados de señal y cierre, o `{ kind: "background", jobId }`; el renderizado nativo aplica el tope de presentación anterior.

#### Efecto de tokens

Los resultados de texto único propiedad del terminal o producidos por política dependen de los datos y están acotados por `maxResultBytes`; una política que sustituye deliberadamente contenido estructurado en varios bloques es dueña del tope de ese contenido. Cada resultado devuelto permanece en el historial hasta la compactación.

#### Efecto de caché KV

De solo añadido; los resultados nuevos siguen el prefijo de petición reutilizable.

## Limitaciones conocidas y trabajo diferido

- No se expone ningún schema de secuencia de teclas con nombre, TUI, BEL, redimensionado, auto-arranque o compartición entre agents.
- El modo en segundo plano requiere tanto `@deepseek-ai/dsh-jobs` como su controlador orientado al modelo.
