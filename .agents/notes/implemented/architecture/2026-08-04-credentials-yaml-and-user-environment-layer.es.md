# Agent Note: Separar el almacén de credenciales de la capa de entorno del usuario

Status: implemented

[English](2026-08-04-credentials-yaml-and-user-environment-layer.md) | Español

## Problema

`$DSH_HOME/.env` cargaba con dos trabajos incompatibles. Era el almacén de secretos escribible de [`credentials-local`](../../../../packages/credentials/credentials-local/README.es.md), así que ninguna superficie podía izarlo a `process.env` — izarlo haría que cada clave almacenada se leyera como una anulación de lanzamiento de solo lectura y bloquearía la rotación desde la página Models. Pero su nombre y su formato dotenv prometen un archivo de entorno, de modo que los usuarios metían en él valores que no son secretos y esos valores no llegaban a ninguna parte: un `DEEPSEEK_BASE_URL` junto a un `DEEPSEEK_API_KEY` que funcionaba en el mismo archivo se ignoraba en silencio, porque solo el provider de credenciales lee el documento y este únicamente direcciona referencias de credenciales.

Un archivo no puede ser a la vez un almacén que el Harness posee y aísla y una capa que se propaga según las reglas ordinarias del entorno. La [decisión de credenciales a nivel de petición](2026-07-29-request-level-llm-config-credentials.es.md) eligió dotenv para alinearse con el `.env` del home de los productos homólogos, y la confusión no se hizo visible hasta que un valor no secreto necesitó el mismo archivo.

## Decisión

Los dos trabajos se convierten en dos archivos bajo el home del Harness.

**`.credentials.yaml` es el almacén gestionado por el provider.** Un mapeo YAML estricto de `CredentialRef` a cadena no vacía, sin campo `version` ni nivel de envoltura:

```yaml
DEEPSEEK_API_KEY: sk-…
OPENAI_API_KEY: sk-…
```

Como el documento contiene credenciales y nada más, cualquier desviación es un rechazo, no una entrada omitida: una raíz que no es un mapeo, una clave que no es un identificador POSIX, un valor que no es una cadena, una cadena vacía, una clave duplicada y un YAML malformado fallan todos — de forma ruidosa al arrancar y al escribir, y con aviso que conserva la última instantánea buena en una recarga en vivo. Una clave ignorada en silencio se leería como «el secreto que almacené no surte efecto», que es justo el fallo que este cambio existe para eliminar. El editor físico de líneas dotenv se sustituye por un parche del documento analizado, de modo que los comentarios y las entradas intactas conservan su formato, cualquier valor de cadena viaja de ida y vuelta (incluidos los multilínea) y ninguna entrada queda sin poder escribirse por falta de un estilo de comillas. El candado del escritor, la lectura-modificación-escritura, la escritura atómica `0600` bajo un directorio `0700`, el vigilante de ruta exacta, la supresión de autoescritura por igualdad de contenido y la disposición en reposo no cambian.

**`$DSH_HOME/.env` es la capa de entorno ordinaria del usuario.** `loadLayeredEnv`, en [`dsh-app-boot`](../../../../packages/boot/app-boot/README.es.md), analiza el `.env` del directorio invocante y después el del home del Harness, con el orden `user < project < inherited` al materializar cada valor aceptado solo cuando el proceso no tiene ya un valor de capa superior. El home del Harness se resuelve desde el entorno heredado *antes* de que se cargue cualquiera de los dos archivos, así que un `.env` de proyecto no puede redirigir qué documento de usuario se lee. Solo el CLI del producto superpone estos archivos; los bins del SDK y de los ejemplos siguen cargando su propio directorio mediante `loadEnv` y no deben heredar el `$DSH_HOME` de un desarrollador.

La precedencia de credenciales distingue el entorno heredado de los archivos descubiertos: el valor heredado sigue siendo la anulación de solo lectura por ejecución, gana después el documento gestionado, y los valores del `.env` de proyecto y luego del de usuario siguen siendo fallbacks escribibles. Por eso un `set` sustituye un valor de archivo descubierto en lugar de rechazar una escritura que solo la vista plana de `process.env` consideraría ensombrecida.

No hay migración. Una clave que ya esté en `$DSH_HOME/.env` sigue resolviéndose como fallback, mientras que el documento gestionado gana en cuanto la página Models almacena esa referencia.

## Consecuencias

- Se renuncia a: una clave dejada en `$DSH_HOME/.env` se materializa en `process.env`, así que llega a los subprocesos bajo la [limpieza de credenciales de subproceso](../../../../packages/subprocess/subprocess/README.es.md) en lugar de permanecer dentro del provider. Sigue siendo un fallback escribible por debajo de `.credentials.yaml`; un secreto que el Harness debe poseer y aislar pertenece al documento gestionado, que nunca se materializa.
- Se gana: un valor no secreto en el `.env` del usuario por fin surte efecto, que era el defecto original; el formato del documento puede rechazar lo que no puede servir; y `0600` cubre un archivo que solo contiene secretos en lugar de un archivo donde se dice a los usuarios que pongan configuración ordinaria.
- El `0600` que escribe el provider también se impone sobre lo que lee: en POSIX, un documento con cualquier bit de permiso de grupo u otros falla el lanzamiento antes de que se lean sus contenidos, al arrancar y en cada recarga, y el diagnóstico nombra la reparación `chmod 600`. Windows no tiene un modo que inspeccionar — sus ACL no son expresables aquí —, así que la comprobación se omite en lugar de fingirse.
- La frontera `0600` sigue deteniendo a otros usuarios del sistema operativo y no al modelo, sin cambios por esta separación — el [README del provider](../../../../packages/credentials/credentials-local/README.es.md) es el dueño de ese límite y de la remisión al provider de llaveros.

## Alternativas consideradas

**Mantener un único `$DSH_HOME/.env` y enseñar al CLI a izarlo.** Rechazada: izar el almacén es exactamente lo que hace que las claves almacenadas no se puedan rotar, que es por lo que [app-boot documentó la exclusión](../../../../packages/boot/app-boot/README.es.md) en primer lugar. El conflicto son los dos trabajos del archivo, no el loader.

**`$DSH_HOME/.credentials.env` — un segundo archivo dotenv.** Rechazada: dotenv sirve para una capa de entorno, pero no puede expresar «un documento gestionado indexado por referencia de credencial». No puede rechazar una clave que no sea cadena ni una clave no direccionable, y su editor de líneas ya rechazaba los valores que no podía entrecomillar, dejando entradas legibles pero no escribibles.

**Añadir un campo `version` al nuevo documento.** Rechazada: el formato es un único mapeo de cadenas restringido por schema, sin variante histórica que discriminar. Mientras el producto no esté publicado, cambiar la estructura y rechazar la anterior es mejor que prometer un protocolo de migración.

**Migrar en el primer arranque las claves con forma de credencial fuera de `$DSH_HOME/.env`.** Rechazada: el código de migración convierte un formato efímero en una superficie de mantenimiento duradera, y clasificar qué claves de un archivo desconocido son secretos es exactamente la ambigüedad que esta separación elimina. El archivo antiguo sigue funcionando como entorno, que es un resultado veraz y no uno silencioso.

**Eliminar por completo la capa de `.env` del usuario y conservar solo el entorno heredado.** Rechazada aquí por estar fuera de alcance: es un diseño coherente (menos capas, un lugar por valor), pero elimina un flujo de trabajo que los usuarios tienen, y la cuestión de las capas pertenece a la decisión de precedencia diferida, no a esta separación.
