# dsh-credentials-local

[English](README.md) | Español

Provider de [credenciales](../credentials/README.es.md) respaldado por archivos: cuatro capas, una precedencia honesta.

| Capa | Id de origen | Escribible | Gana |
|---|---|---|---|
| Entorno de proceso heredado | `env` | no | siempre |
| Documento `$DSH_HOME/.credentials.yaml` | `file` | sí (`set`/`unset`) | sobre ambas capas `.env` |
| `<invocation cwd>/.env` | `project-env` | no aquí | sobre el `.env` de usuario |
| `$DSH_HOME/.env` | `user-env` | no aquí | en caso contrario |

El entorno de lanzamiento gana porque una sobreescritura por ejecución (`DEEPSEEK_API_KEY=… dsh`, un secreto de CI, un `-e` de contenedor) es la intención del operador para esa ejecución — y porque, al no poder editarse desde dentro, debe ser *visiblemente* de solo lectura: `describe()` informa `source: 'env', writable: false`, y `set`/`unset` rechazan en lugar de escribir un cambio que el lector nunca vería.

Todo lo que queda por debajo pierde frente al almacén gestionado, así que una clave escrita desde la página de Models surte efecto de inmediato incluso cuando una clave más antigua está en un `.env`. Esas dos capas siguen resolviendo cuando no hay nada almacenado, y `describe()` las nombra `project-env` o `user-env` con `writable: true` — almacenar una clave las sustituye como fuente efectiva.

Bajo el CLI de producto, la resolución lee la [instantánea de entorno](../../util/launch-environment/README.es.md) congelada del lanzador en lugar de `process.env`: solo la instantánea puede decir si un valor vino del shell de lanzamiento o de un archivo. Una composición que el CLI de producto no arrancó tiene el entorno heredado como única capa, lo que mantiene a los integradores en la semántica que ya tenían.

## Config

| Campo | Valor por defecto | Significado |
|---|---|---|
| `path` | `<harness home>/.credentials.yaml` | Ubicación del documento de credenciales. |
| `dshHome` | `$DSH_HOME` o `~/.dsh` | Home del harness usado cuando se omite `path`. |
| `watch` | `true` | Publicar en caliente las ediciones externas. |
| `debounceMs` | `100` | Ventana de asentamiento de escrituras del watcher. |

## El documento

Un documento YAML versionado con una sección por espacio de claves y nada más:

```yaml
version: 1

refs:
  DEEPSEEK_API_KEY: sk-…
  OPENAI_API_KEY: sk-…

records:
  llm-pi-ai/openai-codex:
    kind: grant
    payload:                    # written verbatim; this provider does not interpret it
      type: oauth
      access: eyJhbGciOi…
      refresh: rft_9f8e7d…
      expires: 1786000000000
  llm-pi-ai/amazon-bedrock:
    kind: api-key               # environment values, no key: this route uses an AWS profile
    env:
      AWS_PROFILE: prod
  llm-pi-ai/amazon-bedrock-dev:
    kind: api-key               # neither: the owner confirmed the ambient credential chain
```

El documento contiene solo credenciales, así que toda desviación es un rechazo y no una entrada omitida — una clave ignorada en silencio se leería como «la credencial que almacené no tiene efecto». Una raíz que no es un mapeo, una clave de nivel superior desconocida, una clave que no es direccionable en su espacio, un valor de tipo incorrecto, una cadena vacía, una etiqueta o un campo de registro desconocidos, una clave duplicada y un YAML malformado: todo falla — con estrépito al arrancar, y con aviso conservando la última instantánea buena en una recarga en vivo.

Un payload de `grant` debe sobrevivir a un viaje de ida y vuelta por JSON, exigencia que se aplica en ambas direcciones. YAML escribe valores que JSON no tiene — `.inf`, ciclos de alias — y un propietario puede entregar un `Date` o un `bigint`; en cualquier caso el almacén rechaza antes que guardar algo que no pudiera leer de vuelta exactamente como se escribió.

El formato anterior al lanzamiento era un mapeo plano sin `version`. Un arranque que lo reconoce exactamente — nombres direccionables sobre escalares de cadena no vacíos, sin directivas — actualiza el documento en su sitio bajo el candado de escritura: las líneas originales quedan anidadas verbatim bajo `refs:`, así que valores, comentarios y grafías sobreviven byte a byte. Cualquier otra forma plana se rechaza por nombre, con el recuento de entradas y la única edición necesaria (`version: 1`, anidar bajo `refs:`) — nunca se lee como un almacén vacío, lo que se manifestaría como un fallo de autenticación en la primera petición en lugar de en la carga. Una recarga en vivo nunca migra: un documento plano restaurado a mitad de ejecución conserva la última instantánea buena hasta el siguiente arranque.

Las escrituras modifican el documento analizado en lugar de reconstruirlo, así que los comentarios y el formato de cada entrada intacta sobreviven. Un comentario directamente encima de una entrada es la anotación de esa entrada y se elimina con ella. Cada escritura vuelve a leer primero el documento bajo el candado de escritura entre procesos de [`dsh-atomic-write`](../../util/atomic-write/README.es.md) y publica cualquier cosa que no hubiera observado; después confirma atómicamente con modo `0600` bajo un directorio solo del propietario (`0700`) — de modo que un escritor concurrente o una edición externa dentro de la ventana de debounce del watcher se integra en lugar de sobrescribirse. Un documento en disco que ya no se analiza hace fallar la escritura en lugar de sobrescribir contenido que el provider no pudo entender.

Cualquier valor de cadena sobrevive a la ida y vuelta, incluidos los valores multilínea, así que ninguna entrada es inescribible por falta de un estilo de comillas. Un valor almacenado vacío está ausente, según la regla del seam — que es por lo que una cadena vacía en el documento se rechaza sin más: `unset` elimina una clave, no la deja en blanco.

## Permisos

El provider crea el directorio `0700` y crea o sustituye atómicamente el documento `0600`. Mantiene lo que *lee* dentro de ese mismo límite: en POSIX, un documento con cualquier bit de grupo o de otros falla antes de que se analice su contenido — al arrancar y en cada recarga — y el error nombra la reparación `chmod 600`. Windows no tiene modo que inspeccionar, así que allí la comprobación se omite en lugar de fingirse.

## Recarga en caliente

Las ediciones externas publican `credentials/reference-updated` por cada referencia modificada después de que la instantánea se sustituya **en bloque** — una entrada eliminada en disco nunca permanece en memoria. Antes de que Chokidar abra el objetivo, el provider resuelve el realpath de su ancestro existente más profundo y restaura cualquier sufijo que falte; el acceso a archivos y los diagnósticos conservan la ruta configurada, mientras que Windows no puede mezclar un alias 8.3 con eventos libuv de formato largo. Las escrituras propias del provider se reconocen por contenido y publican exactamente su único evento de commit. Un documento ilegible o inválido en tiempo de ejecución conserva la última instantánea buena y avisa; un archivo ausente es un almacén vacío; un archivo ilegible o inválido al arrancar falla con estrépito.

## Límite de seguridad

El documento es `0600` bajo un directorio `0700`, lo que frena a otros usuarios del SO — **no** al modelo. Los procesos de herramientas (bash, las herramientas del sistema de archivos) se ejecutan como el mismo usuario, y la política de archivos `workspace-write` incluida confina las mutaciones, no las lecturas, así que pueden leer este archivo exactamente como cualquier otro archivo que el usuario posea; ningún modo de sandbox lo distingue. Lo que el harness sí mantiene es más estrecho: nunca entrega al modelo una ruta resuelta al documento y nunca lo carga en el entorno del proceso — a diferencia de `$DSH_HOME/.env`, que es la capa de entorno ordinaria del usuario (véanse las [capas de Harness-home de app-boot](../../boot/app-boot/README.es.md#profiles)) — así que llegar al valor exige una lectura deliberada de una ruta que no se le dio al agent.

Eso es discreción, no un límite. Un despliegue que deba mantener las claves de los providers alejadas de su propio agent no puede conseguirlo con permisos de archivo; un provider de llavero del SO — un almacén que los procesos del modelo no pueden leer en absoluto — es la respuesta aplazada y debe vivir junto a este provider como paquete hermano.

## Experiencia del modelo

Indirectamente, a través de los adaptadores LLM consumidores: los valores almacenados autorizan sus peticiones a los providers, y el adaptador es dueño de toda superficie visible para el modelo.

#### Efecto en la KV Cache

Sin invalidación directa; las credenciales nunca entran en un prefijo de petición.

## Limitaciones conocidas y trabajo pendiente

- **Las escrituras concurrentes sobre la misma referencia ganan por última escritura** — el candado de escritura y la lectura-modificación-escritura evitan que los escritores concurrentes se eliminen entradas entre sí, pero dos escritores que editan una misma referencia siguen resolviendo a favor de la escritura posterior; no hay comprobación de revisión.
- **Un proceso con el mismo UID puede leer el documento** — véase [Límite de seguridad](#security-boundary): los modos de sandbox de efecto en archivos no deniegan lecturas, y un provider de llavero del SO queda aplazado.
- **Los cambios de entorno son invisibles** — la instantánea se congela en el lanzamiento, así que una variable exportada después del arranque no llega ni a la resolución ni a `describe`; cambiar una credencial proveniente del entorno exige reiniciar.
- **Atómico, pero no durable frente a caídas** — heredado de `dsh-atomic-write`; el almacén vuelve a leer en el arranque.
