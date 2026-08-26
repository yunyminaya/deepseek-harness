# dsh-atomic-write

[English](README.md) | Español

Reemplazo atómico de archivos sin dependencias compartido por los almacenes respaldados por archivos que nunca deben dejar en disco contenido parcial, secuestrado por symlink ni más amplio de lo previsto — el documento de ajustes de usuario (`dsh-settings-file`) y el almacén de credenciales (`dsh-credentials-local`).

## Superficie

```ts
import { withFileLock, writeFileAtomic } from '@deepseek-ai/dsh-atomic-write'

declare const text: string
declare const render: (previous: string) => string

await writeFileAtomic('/home/u/.dsh/settings.yaml', text, { mode: 0o600 })

// Read-modify-write against the same file from several processes.
await withFileLock('/home/u/.dsh/settings.yaml', async () => {
  await writeFileAtomic('/home/u/.dsh/settings.yaml', render(text), { mode: 0o600 })
})
```

`writeFileAtomic` confirma una cadena ya renderizada. El contrato, en el orden en que los fallos podrían explotarlo:

- **Temp de creación exclusiva** (`wx`, sufijo aleatorio): la apertura se niega a seguir un symlink plantado en una ruta temp adivinable.
- **El inode nuevo transporta `mode` a través del rename**: reemplazar un archivo con permisos más amplios los estrecha sin una carrera de chmod. `mode` es obligatorio para que la decisión de permisos siga visible en cada punto de llamada (sujeta al umask del proceso, como todo inode nuevo).
- **`rename` reemplaza el propio destino con symlink**, sin escribir nunca a través de él hasta su referente.
- **Hermano en el mismo directorio** mantiene el rename en un solo sistema de archivos, de modo que el intercambio sigue siendo atómico.

Se crean los directorios padre; ante cualquier fallo se elimina el temp y se relanza el fallo; los lectores observan el contenido completo antiguo o el nuevo.

`withFileLock` serializa los escritores de un archivo entre procesos, para los ciclos de leer-renderizar-confirmar que un commit atómico desnudo no puede hacer seguros por sí solo. El bloqueo es un hermano `<filename>.lock` creado con `wx`, por lo que los lectores nunca compiten; los que esperan retroceden exponencialmente y fallan con un timeout en lugar de bloquearse para siempre. `EEXIST` identifica la contención directamente; `EPERM` solo lo hace cuando un `lstat` reciente confirma que la ruta del bloqueo existe, cubriendo el comportamiento de creación exclusiva de Windows sin ocultar un fallo de permisos no relacionado. Un contendiente nunca elimina el bloqueo existente: la antigüedad no puede distinguir un propietario que se estrelló de un escritor vivo en pausa.

Cuánto espera un contendiente es una propiedad de la operación que ejecuta el titular, por lo que se declara por llamada mediante `waitMs`. El valor por defecto está dimensionado solo para trabajo de archivos; un titular cuyo ciclo incluya un ida y vuelta de red — una mutación de credenciales que refresca un token caducado — declara uno mayor, porque dejar el valor por defecto haría fallar a todos los demás escritores de ese archivo durante ese tiempo. La cadencia de reintento permanece fija: gobierna con qué frecuencia pregunta un contendiente, algo que ningún llamador tiene motivos para variar.

## Experiencia del modelo

Ninguna, ya que es una primitiva de sistema de archivos pura; nada de esto llega a una solicitud de modelo.

#### Efecto de KV Cache

Ninguno; nada de esto entra en un prefijo de solicitud.

## Limitaciones conocidas y trabajo diferido

- **Atómico, no duradero** — sin `fsync` del archivo ni de su directorio, por lo que tras un crash puede observarse el rename deshecho. Los almacenes respaldados por archivos de aquí releen y republican al arrancar, dejando la durabilidad como política del llamador.
- **Solo contenido de cadena** — sin forma `Buffer` ni de flujo hasta que un consumidor la necesite.
- **Los bloqueos huérfanos requieren recuperación del operador** — un proceso que sale mientras mantiene el bloqueo puede dejar el hermano atrás. Los escritores posteriores agotan su timeout sin eliminarlo; un operador lo elimina solo tras verificar que ningún escritor sigue siendo dueño de él. La antigüedad del archivo por sí sola no es una prueba segura de abandono.
