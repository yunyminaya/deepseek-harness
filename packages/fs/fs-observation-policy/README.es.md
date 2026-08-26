# @deepseek-ai/dsh-fs-observation-policy

[English](README.md) | Español

El **plugin fs-observation-policy**: registra la presencia o ausencia observada y añade lectura-antes-de-editar más escritura/edición protegida sobre el contrato de provider `ctx.fs` ([`@deepseek-ai/dsh-fs`](../fs)) — a través de la compuerta de eventos `fs/*`, **NO** a través de un servicio de métodos. Este plugin no registra **ningún** servicio `ctx.fsPolicy` y no tiene métodos públicos `read`/`write`/`edit`/`resolve`. Es la parte de política de la pila de sistema de archivos: no un seam sustituible, sino la política que no pertenece a la clase base del provider `FileSystem`.

```ts
import type { Context } from '@deepseek-ai/cordis'
import * as FsPolicy from '@deepseek-ai/dsh-fs-observation-policy'

declare const ctx: Context

// No service to inject — this plugin only registers the three fs/* listeners.
// Load it alongside a ctx.fs provider (e.g. @deepseek-ai/dsh-fs-local) and the
// @deepseek-ai/dsh-tool-fs tools; the tools dispatch the fs/* events this plugin
// decides. Order does not matter for resolution (no inject), but the policy
// listener should be the first decider registered for the fs/*-intent slots.
await ctx.plugin(FsPolicy)
```

## La división en cuatro capas

| Capa | Paquete | Rol |
|---|---|---|
| herramienta / ejecutor | `@deepseek-ai/dsh-tool-fs` | schemas orientados al modelo + ventana de lectura + renderizado de texto; lecturas/escrituras/ediciones mediante `ctx.fs`, despacha los eventos `fs/*` |
| política | `@deepseek-ai/dsh-fs-observation-policy` (este) | estado observado + lectura-antes-de-editar + escritura/edición protegida por versión, contribuida a través de la compuerta de eventos `fs/*` (sin servicio) |
| contrato de provider | `@deepseek-ai/dsh-fs` | `ctx.fs`: E/S de texto + primitivas de mutación atómica (guardia de versión opcional); es propietario del vocabulario de eventos `fs/*` |
| provider | `@deepseek-ai/dsh-fs-local` | implementación local de `ctx.fs` |

## Cómo participa la compuerta

Tres eventos `fs/*` (declarados por `@deepseek-ai/dsh-fs`, despachados por `@deepseek-ai/dsh-tool-fs`):

| Evento | Listener de este plugin |
|---|---|
| `fs/write-intent` | No visto u observado ausente → `{ kind: 'createIfAbsent' }`; observado presente → `{ kind: 'replaceIfVersion', version: vObserved }`. Decisión de un solo slot; NO llama a `next()`. |
| `fs/edit-intent` | No visto → `FS_NOT_OBSERVED`; observado ausente → `FS_NOT_FOUND`; observado presente → `{ version: vObserved }` como base de CAS. Decisión de un solo slot; NO llama a `next()`. |
| `fs/observed` | Registra `{ kind: 'present', version }` o `{ kind: 'absent' }` para este propietario+objetivo. `WeakMap.set` síncrono, solo de efectos secundarios. |

## El estado observado es el registro de observaciones previas; la frescura es CAS del provider

El estado observado es un mapa débil de propietario a objetivo con tres estados lógicos: no visto, ausencia confirmada o presente en una versión. Una lectura o mutación de archivo exitosa registra presencia; un fallo de metadatos de `read` o del comando `view`, `str_replace` o `insert` de `str_replace_editor` registra ausencia antes de devolver `FS_NOT_FOUND`. El plugin no realiza E/S de sistema de archivos: convierte ese estado en una guardia del provider. La presencia aporta la versión observada, mientras que la ausencia solo permite que proceda una escritura `createIfAbsent`; la edición no tiene base de versión y devuelve `FS_NOT_FOUND`. Una lectura con ventana observa la versión completa del archivo, de modo que una edición dirigida posterior solo se permite mientras ese archivo permanezca sin cambios. El estado se descarta al eliminar el plugin y no se persiste entre sesiones.

## Un solo slot, gana el primero

Los slots `fs/write-intent`/`fs/edit-intent` contienen exactamente un decisor — este plugin decide por completo y no llama a `next()`. El slot es de gana-el-primero por orden de registro; que este plugin lo posea es la convención de despliegue por defecto, no un invariante impuesto por eventos (un decisor registrado antes / antepuesto ganaría en su lugar). Esto no es una cadena de autorización componible — la interceptación en capas de permiso/auditoría/sandbox corresponde a `tools/execute`.

## Sin acoplamiento por métodos

Como el plugin influye en el mundo solo a través de eventos, quitarlo no rompe `@deepseek-ai/dsh-tool-fs` en un límite de inyección de servicios: la herramienta cae al provider `ctx.fs` desnudo (escritura/edición incondicional, sin estado observado). Volver a cargarlo superpone la política. Esa adición/eliminación elegante es todo el sentido de la compuerta de eventos frente a un servicio de métodos obligatorio.

## Experiencia del modelo

### Resultado de la herramienta de sistema de archivos

#### Lo que ve el modelo

Este plugin no añade prompt ni schema. Rechaza una edición sin una observación previa con el código `FS_NOT_OBSERVED` y el mensaje exacto `edit requires reading "<path>" first`; editar un objetivo recién observado ausente devuelve `FS_NOT_FOUND`. Las mutaciones protegidas cuya observación positiva está obsoleta propagan el error `FS_STALE_VERSION` propiedad del provider. [`dsh-tool-fs`](../tool-fs/README.es.md) es propietario del envoltorio de errores orientado al modelo, que añade la instrucción de recuperación a los mensajes `FS_STALE_VERSION` (— vuelve a leer el archivo y reintenta) y `FS_NOT_OBSERVED` (— lee el archivo y reintenta) conservando el código. Aplicar el remedio de obsolescencia sobre un objetivo eliminado externamente ahora registra ausencia: la siguiente escritura protegida puede recrearlo con `createIfAbsent`, mientras que el provider preserva atómicamente cualquier creador concurrente.

#### Efecto en tokens

Cero tokens en las operaciones permitidas más allá del resultado normal de la herramienta. Una denegación añade el pequeño resultado de error retenido y evita cualquier carga útil de éxito.

#### Efecto en la caché KV

Solo anexión; el contenido recién visible sigue el prefijo de solicitud reutilizable y no invalida las entradas existentes de la caché KV.

## Limitaciones conocidas y trabajo diferido

- **El estado observado no sobrevive a la reanudación de una sesión** — la persistencia del registro `WeakMap` está diferida, de modo que una sesión reanudada debe volver a leer los archivos antes de escrituras/ediciones protegidas.
- **Los actores sin sesión de agent (agente) nunca pueden satisfacer la política** — sus ediciones lanzan `FS_NOT_OBSERVED` y sus escrituras siempre resuelven `createIfAbsent`, de modo que un llamador no-agent no puede sobrescribir un archivo existente a través de la compuerta.
- **Las lecturas directas de `ctx.fs` no emiten `fs/observed`** — un archivo leído fuera de la herramienta `read` permanece sin observar, y una edición protegida posterior lo rechaza con `FS_NOT_OBSERVED` hasta que la herramienta lo lea.
- **La autorización es frescura de versión, no integridad de vista** — cualquier lectura con ventana autoriza una sobrescritura de archivo completo de un archivo sin cambios, deliberadamente más débil que una regla de vista completa ([Agent Note de división del seam](../../../.agents/notes/implemented/simplification/2026-06-26-fsspec-style-fs-seam.es.md)).
