# @deepseek-ai/dsh-settings-file

[English](README.md) | Español

Provider de ajustes respaldado por archivo. Un único documento YAML o JSON transporta todas las secciones de namespaces; las ediciones externas se publican en caliente a través de `ctx.settings`, y `update()` vuelve a leer el documento bajo un candado de escritor antes de escribir de vuelta de forma atómica, preservando los comentarios YAML del usuario, cualquier sección propiedad de un plugin que no esté cargado en ese momento y cualquier cambio en disco que este proceso aún no haya observado.

## Configuración

| Campo | Significado | Predeterminado |
|---|---|---|
| `path` | Ruta del documento de ajustes; la extensión decide el formato (`.yaml`/`.yml`/`.json`) | `settings.yaml` bajo el home del harness |
| `dshHome` | Home del harness usado cuando se omite `path` | `$DSH_HOME` o `~/.dsh` |
| `watch` | Observa el documento y publica en caliente las ediciones externas | `true` |
| `debounceMs` | Ventana de asentamiento de escritura del watcher en milisegundos | `100` |

El valor predeterminado es un paso explícito `resolveSpec(config)`; una extensión no soportada falla en la carga.

## Comportamiento

- **El arranque falla en voz alta, la recarga conserva el último valor bueno.** Un documento existente pero inválido hace fallar la carga del plugin; una vez en vivo, una edición ilegible o no analizable avisa y conserva las últimas secciones buenas. Un documento ausente resuelve cada namespace desde los valores predeterminados y `base`; su eliminación publica ese mismo estado vacío.
- **Cada escritura es un read-modify-write.** Una persistencia primero vuelve a leer el documento y publica cualquier diferencia en el seam — una edición externa aún dentro de la ventana de debounce del watcher, un cambio que el watcher no captó o una escritura de otro proceso — y luego renderiza contra ese texto fresco, de modo que una escritura nunca puede resucitar un documento obsoleto ni soltar una sección hermana no observada. Si el documento en disco se volvió inválido, la escritura rechaza en voz alta en lugar de sobrescribir la edición manual del usuario.
- **Las escrituras mantienen un candado de escritor entre procesos.** El ciclo leer-renderizar-renombrar se ejecuta bajo un hermano `<file>.lock` creado con `wx`, con backoff exponencial y un plazo de adquisición de 2 s. Un contendiente agota el tiempo sin eliminar el candado existente porque la antigüedad no puede distinguir un propietario estrellado de un escritor vivo en pausa; la recuperación de huérfanos es una acción de operador. Los lectores nunca toman el candado: el commit de rename es atómico, así que las recargas son siempre consistentes.
- **La escritura de vuelta es atómica, solo del propietario y a prueba de symlinks.** El renderizado crea en exclusiva un hermano temporal de sufijo aleatorio con modo `0600` (`wx` se niega a seguir un symlink plantado) y renombra sobre el destino, limpiando el temporal en caso de fallo.
- **Las ediciones YAML son diffs a nivel de hoja.** Una escritura fija solo los valores que cambiaron y elimina solo las claves que se quitaron, de modo que los comentarios, anclas y formatos sobreviven en cada nodo intacto y en la clave de cada par cambiado; un array cambiado (u otro valor que no sea mapa) se reemplaza por completo, llevándose los comentarios que contiene. El JSON se vuelve a serializar sin comentarios.
- **Las recargas y las escrituras comparten una única cadena de operaciones.** Los refrescos del watcher y las persistencias de cada namespace se ponen en cola y se ejecutan de uno en uno en orden de cola; cada renderizado ve el texto que confirmó la operación anterior.
- **La señal de ready del watcher reconcilia una vez.** La carga inicial compite con el propio setup del watcher, de modo que un cambio escrito en medio nunca dispara un evento; la reconciliación en ready cierra esa brecha de arranque.
- **El watcher nativo recibe una ruta canónica.** Antes de que Chokidar abra el destino, el provider resuelve la ruta real (realpath) de su ancestro existente más profundo y restaura cualquier sufijo ausente. El acceso a archivos y los diagnósticos orientados al usuario conservan la ruta configurada, mientras que Windows no puede mezclar un alias 8.3 con rutas de evento largas dentro de libuv.
- **La disposición se aquieta en todo modo de observación.** El teardown marca el provider como cerrado, cierra el watcher cuando existe y luego espera a que termine cada operación de documento en cola o en curso, de modo que nada se publica después de la disposición.
- **Supresión de la autoescritura por contenido.** El provider guarda en caché el último texto bueno; un evento del watcher cuyo contenido coincide con la caché (incluida su propia escritura) es un no-op.
- **Los adaptadores de configuración del Host reciben la ruta resuelta.** `ctx.settings.documentPath` es el nombre de archivo absoluto de `resolveSpec()`, incluida una ruta YAML/JSON personalizada; `prepareDocument()` preserva un archivo existente o crea en exclusiva un archivo vacío ausente con permisos solo del propietario antes de que el Host lo abra. El navegador recibe solo un indicador de disponibilidad, nunca reconstruye `$DSH_HOME` ni envía un destino del sistema de archivos.

## Model Experience

De forma indirecta, a través de los consumers de `ctx.settings`: este provider solo almacena y publica secciones de namespaces, y la surface propia de cada consumer documenta cualquier efecto en el modelo.

#### Efecto de KV Cache

Sin invalidación directa; el plugin consumidor es dueño de cualquier cambio en el prefijo de petición.

## Limitaciones conocidas y trabajo diferido

- **Los conflictos del mismo namespace siguen siendo last-write-wins** — el candado de escritor y el read-modify-write impiden que escritores concurrentes se eliminen los namespaces entre sí, pero dos escritores que editan un mismo namespace siguen resolviendo a la escritura posterior; no hay fusión por valor ni comprobación de revisión.
- **Un evento de watcher perdido permanece invisible hasta la siguiente señal** — las lecturas nunca vuelven a hacer stat del archivo, así que un cambio que el watcher no consigue informar solo se incorpora con el siguiente evento, la siguiente escritura o un reinicio.
- **La preservación de comentarios es solo YAML y con forma de mapa** — los documentos JSON se vuelven a serializar sin comentarios (JSON no tiene ninguno), y los comentarios dentro de un array cambiado (o pegados en línea a un valor escalar cambiado) se van con el valor que describían.
- **Sin indirección de valores** — las secciones contienen valores literales; las referencias estilo `${env:VAR}` para secretos son una funcionalidad diferida a nivel de seam.
