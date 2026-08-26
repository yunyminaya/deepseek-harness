# dsh-credentials

[English](README.md) | Español

Service Definition de credenciales (`ctx.credentials`). Una doctrina, tres consecuencias:

**La configuración lleva referencias a los secretos, nunca los secretos.** Una sección de ajustes o una entrada de `cordis.yml` dice `apiKeyEnv: DEEPSEEK_API_KEY`; el valor detrás de esa referencia vive con un provider de credenciales. Así, el documento de ajustes sigue siendo seguro de sincronizar y de mostrar en una UI de configuración, `describe()` puede responder «¿está configurado, de dónde viene, puedo escribirlo?» sin tener nunca un valor entre manos, y rotar un secreto no toca ningún archivo de configuración.

**Los Consumers resuelven por operación.** `resolve(ref)` se llama al inicio de cada operación (los adaptadores LLM resuelven una vez por petición al modelo) y nunca se cachea entre operaciones — esa lectura es lo que hace que una credencial modificada llegue a la siguiente petición sin reiniciar ningún plugin.

**Un valor almacenado vacío está ausente.** En todas partes: `resolve` lo omite, `describe` lo informa como no configurado. Un valor en blanco nunca puede hacerse pasar por un secreto configurado.

## Dos espacios de claves, dos preguntas

Un `CredentialRef` responde *qué hay detrás de este nombre de variable de entorno*, en capas sobre el entorno del proceso, el almacén gestionado y los archivos `.env`. Todo lo anterior describe esa mitad.

Un `CredentialKey` responde *qué credencial guarda este plugin para este id*. Aquí nada puede apilarse en capas — una concesión de autorización no tiene entorno del que leerse — así que la presencia del registro es todo el hecho, y la regla del valor vacío no se aplica: un registro `api-key` que no lleva ni clave ni valores de entorno declara que su propietario confirmó la autenticación ambiental, lo cual cuenta como configurado.

La clave es `<scope>/<id>`, donde `scope` es el **nombre registrado del plugin propietario**. El scope es el propietario y no el dominio porque un payload de `grant` se escribe en el formato de su propietario: dos plugins que sirvieran el mismo nombre de provider leerían el payload del otro, y un registro dejado por un plugin desinstalado no podría distinguirse de uno vivo. La `/` mantiene además disjuntas las dos gramáticas, así que los espacios de claves nunca pueden colisionar. Un consumer cuyo id llega de otro sitio — una clave de un diccionario de ajustes, el id de provider propio de una biblioteca — pregunta a `isCredentialKeySegment` antes de construir una clave, porque un id fuera de la gramática nunca pudo haber almacenado un registro y debe leerse como «nada almacenado» en lugar de lanzar un error sobre la dirección.

## Superficie

```ts
import type { Context } from '@deepseek-ai/cordis'
import { credentialKey, credentialRef } from '@deepseek-ai/dsh-credentials'

declare const ctx: Context

const ref = credentialRef('DEEPSEEK_API_KEY')            // POSIX shell identifier, branded
const hit = await ctx.credentials.resolve(ref)           // { value, source } | undefined
const info = await ctx.credentials.describe(ref)         // { configured, source?, writable } — never the value
await ctx.credentials.set(ref, 'sk-…')                   // rejects while a read-only source shadows the ref
await ctx.credentials.unset(ref)                         // no-op when absent; same shadowing rule

const key = credentialKey('llm-pi-ai', 'openai-codex')   // <owner>/<id>, branded
await ctx.credentials.readRecord(key)                    // CredentialRecord | undefined
await ctx.credentials.describeRecord(key)                // { configured, kind?, writable } — never the value
await ctx.credentials.listRecords()                      // [{ key, kind }] — never values
await ctx.credentials.modifyRecord(key, async () => ({ kind: 'grant', payload: { token: '…' } }))
await ctx.credentials.deleteRecord(key)                  // no-op when absent
```

`modifyRecord` es el único camino de escritura porque una escritura correcta depende del valor actual: una renovación de token es leer-decidir-sustituir, y la mutación debe ver el registro tal como está en el momento en que la escritura es exclusiva. La exclusión se mantiene entre procesos, que es lo que impide que dos de ellos roten un mismo refresh token y se pierda el que escribió primero. Devolver `undefined` desde la mutación deja la entrada intacta y no anuncia nada.

`listRecords` existe aunque la mitad de referencias no tenga enumeración por diseño. Las referencias se descubren desde los schemas de ajustes (campos `apiKeyEnv`); los registros no tienen ese camino, así que una superficie que no pueda listarlos no puede mostrar para qué está autorizado un usuario, ni encontrar un huérfano dejado por un plugin desinstalado.

El `payload` de un registro `grant` es opaco: el seam nunca lo lee, lo valida ni le cambia la forma. La única restricción es que sobreviva a un viaje de ida y vuelta por JSON, que el provider aplica a la entrada y a la salida — un valor que el almacén no pudiera leer de vuelta exactamente como se escribió se rechaza en lugar de almacenarse con pérdidas.

`credentials/reference-updated (ref)` se dispara después de un cambio confirmado en una fuente gestionada por un provider — un `set`, un `unset` o una edición externa observada en el almacenamiento. Los cambios ambientales del entorno del proceso no son observables y nunca emiten. Los Consumers no necesitan el evento (vuelven a resolver por operación); existe para que las UIs de configuración refresquen una insignia de «configurado». Su declaración vive en la exportación de la subruta `./types` segura para el cliente, junto con el tipo `CredentialRef` que nombra (la raíz del paquete re-exporta el tipo), de modo que un consumer fuera de la cara de compilación del Host lee exactamente la firma que el Host emite en lugar de reescribirla.

La regla de sombreado en `set`/`unset` es un fallo ruidoso deliberado: cuando una fuente de solo lectura (el entorno vivo del proceso, en el provider local) suministra actualmente la referencia, una escritura parecería tener éxito mientras la resolución sigue devolviendo el valor que la sombrea — el seam rechaza en su lugar, y `describe().writable` permite que una UI muestre la referencia como de solo lectura desde el principio.

## Providers

[`dsh-credentials-local`](../credentials-local/README.es.md) apila el entorno de proceso heredado sobre su documento gestionado `$DSH_HOME/.credentials.yaml`, con las capas `.env` de proyecto y de usuario del lanzador como respaldo. La forma del seam deja sitio para providers respaldados por llavero, por comandos auxiliares y por KMS; un provider de ajustes remoto nunca necesita llevar secretos.

## Experiencia del modelo

Indirectamente, a través de los adaptadores LLM consumidores: un valor resuelto autoriza sus peticiones a los providers, y el adaptador es dueño de toda superficie visible para el modelo.

#### Efecto en la KV Cache

Sin invalidación directa; las credenciales nunca entran en un prefijo de petición.

## Limitaciones conocidas y trabajo pendiente

- **Las referencias no tienen enumeración** — el seam responde preguntas sobre las referencias que se le dan; las superficies de configuración las aprenden de los schemas de ajustes, así que un `list()` sobre esa mitad no tiene consumer actual. Los registros sí se enumeran, por la razón anterior.
- **Las referencias tienen forma de variable de entorno** — un único espacio de nombres plano de identificadores POSIX, porque una referencia hace las veces del nombre de entorno a través del cual se resuelve. Los registros llevan el direccionamiento más rico `<owner>/<id>`.
- **Los cambios del entorno de proceso son invisibles** — ningún evento puede dispararse por ellos; una UI solo vuelve a leer `describe()` en su propia navegación.
- **El propietario de un registro es su scope, y nada verifica que el scope esté montado** — el seam almacena lo que se le da e informa de lo que almacena. Reconocer un huérfano es tarea del llamador, entre `listRecords()` y el registro que sea dueño de ese scope; el seam no tiene un registro propio contra el que comprobarlo.
