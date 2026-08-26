# @deepseek-ai/dsh-fs-e2b

[English](README.md) | Español

Implementación E2B del contrato de provider de [`@deepseek-ai/dsh-fs`](../../fs/fs/README.es.md). No tiene config: carga primero [`@deepseek-ai/dsh-e2b`](../e2b/README.es.md) y después este servicio en lugar de `dsh-fs-local`. El provider usa el cwd remoto y el handle del SDK del propietario, así que las herramientas de archivos observan el mismo mundo que los procesos Bash respaldados por E2B.

## Comportamiento

- **Identidad y metadatos remotos** — las rutas relativas se resuelven como rutas POSIX contra el cwd del llamador o `ctx.e2b.cwd`; GNU `realpath -mz` aporta la identidad canónica del objetivo sin exigir que exista el archivo final, y el enmarcado ASCII/base64 con NUL estricto preserva las rutas con saltos de línea y multibyte a través del transporte decodificado del SDK. `stat`, `lstat` sin seguir enlaces y los listados de directorio estables de un nivel proyectan los metadatos de E2B en el seam del sistema de archivos; los listados reutilizan los metadatos devueltos y resuelven las entradas de symlink secuencialmente. Las versiones son hashes opacos de los metadatos de E2B más un atributo extendido por escritura.
- **Rutas del mundo de ejecución** — los objetivos canónicos exponen rutas POSIX absolutas de procesos, URIs `file:` codificadas en porcentaje y comprobaciones de contención propiedad del provider, de modo que los consumers genéricos de subprocesos nunca analizan ids de objetivo de E2B ni aplican reglas de rutas del host.
- **Lecturas UTF-8** — las lecturas completas y las lecturas en streaming preservan la decodificación entre fragmentos, rechazan el UTF-8 inválido y usan la muestra NUL de 8192 bytes del seam para la detección de binarios. La herramienta orientada al modelo sigue siendo dueña de la selección de tamaño y de las ventanas de líneas.
- **Lecturas de bytes crudos acotadas** — `readBytes` corta en el tamaño del stat antes de cualquier transferencia de contenido, después transmite el objeto remoto en streaming y cancela el stream en el primer fragmento que supera `maxBytes` (`FS_TOO_LARGE`), de modo que ni un archivo sobredimensionado en reposo ni uno que crece después del stat se almacenan enteros en la memoria del host. La peculiaridad del archivo vacío del SDK fijado (content-length 0 devuelve `''` en formato de stream) produce un resultado vacío.
- **Mutaciones atómicas** — las escrituras crean un directorio de staging hermano aleatorio, lo cambian a modo `0700` antes de subir el contenido y preservan el modo POSIX de un archivo existente. Las sustituciones se publican mediante el rename atómico de mismo sistema de archivos de E2B. Un `createIfAbsent` protegido publica en su lugar con `ln -T` remoto, haciendo el commit atómicamente sin-reemplazo incluso cuando aparece un directorio en el destino; los metadatos leídos del archivo en staging antes de ese commit se proyectan a la ruta del objetivo para la versión devuelta, de modo que ninguna petición de metadatos falible sigue a ninguno de los dos puntos de commit. E2B crea los directorios padre que faltan. Las ediciones literales normalizan LF para la coincidencia, restauran el almacenamiento CRLF dominante y serializan las mutaciones por objetivo canónico dentro del proceso del host.
- **Fallos y cancelación** — los fallos de E2B de no-encontrado, permiso, aborto y demás del controlador se mapean al vocabulario `FsError` existente. La cancelación es de mejor esfuerzo en los límites anteriores de las peticiones al SDK y se comprueba inmediatamente antes de la publicación. La señal no se reenvía al commit de rename ni de enlace protegido, así que la cancelación no puede interrumpir la publicación atómica ni convertir una escritura confirmada en un fallo notificado.

El provider no copia, monta ni reconcilia el workspace del host. Pasarle una ruta del host como `cwd` crea un directorio remoto con la misma grafía, nada más.

## Experiencia del modelo

Indirectamente, a través de [`dsh-tool-fs`](../../fs/tool-fs/README.es.md), que renderiza contenido UTF-8 remoto, resultados de directorio, confirmaciones de mutación y errores de provider mientras la identidad y el transporte de E2B siguen siendo internos.

#### Efecto en la KV Cache

Sin invalidación directa; el consumer nombrado es dueño de cualquier cambio en el prefijo de petición.

## Limitaciones conocidas y trabajo pendiente

- **Sin sincronización con el host** — un cwd E2B vacío sigue vacío hasta que una herramienta, un comando o un proceso externo lo pueble; los archivos locales ni se suben ni se reflejan de vuelta.
- **La coordinación de mutaciones es local al proceso del host** — `createIfAbsent` preserva a un creador remoto que compite por la publicación, pero otra conexión o comando del harness aún puede competir por la sustitución; las guardas de versión detectan solo los cambios de metadatos que E2B representa.
- **Las lecturas reabren los objetivos canónicos por ruta** — una sustitución remota concurrente de la ruta entre la resolución y la apertura del stream no está cercada por un handle de archivo estable; ningún defecto de producto observado justifica un protocolo de lectura acotada específico del provider en este POC.
- **Siguen los costes de mutación de archivo completo** — los diffs de sobrescritura y las ediciones literales leen archivos completos a la memoria del host, y cada operación incurre en la latencia del controlador de E2B.
- **El POC apunta a la imagen Linux por defecto de E2B** — depende de GNU `realpath`/`base64`/`chmod`, del rename en el mismo sistema de archivos, de las lecturas en streaming y de los atributos extendidos de metadatos; las plantillas personalizadas quedan fuera de este POC.
