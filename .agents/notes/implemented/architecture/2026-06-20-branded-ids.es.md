# Agent Note: IDs con marca en todos los lugares donde corresponden

Status: implemented

[English](2026-06-20-branded-ids.md) | Español

## Problema

El harness marca `CallId` (`packages/llm/llm/src/brand.ts`) y el `SessionId` compartido de agent/sesión (`packages/core/session/src/types.ts`) con la maquinaria `Branded<B> = string & { readonly [BRAND]: B }` (propiedad del paquete solo de tipos `@deepseek-ai/dsh-brand` en `packages/util/brand/` — ver su [README](../../../../packages/util/brand/README.es.md)) y una fábrica de conversión de costo cero por tipo. `dsh-brand` también declara la política rectora: *«La marca es para los ids que cruzan fronteras de paquete y podrían confundirse de forma plausible; no todo string necesita una marca.»* Esa política es correcta; el problema es que solo se aplica a medias. Dos vacíos permiten hoy que un string estructuralmente idéntico pero semánticamente erróneo se cuele por el comprobador de tipos.

**Vacío 1 — ids sin marca que cruzan fronteras en el seam de bash.** El id de trabajo en segundo plano es un `string` simple: `BashTask.id: string` (`packages/shell/shell/src/types.ts`), transportado como `string` por todo el seam del executor (`ShellExecutor.get`/`ownerOf`/`readOutput`/`kill(id: string)` en `packages/shell/shell/src/index.ts`) y validado y pasado como `string` por las herramientas de cara al modelo (`validateJobId`, `assertTaskAccess`, el argumento de schema `job_id` en `packages/shell/tool-bash/src/index.ts`). Lo genera un contador por executor — `` `bash-${this.nextTaskId++}` `` en `packages/shell/bash-local/src/index.ts` — lo que le da **exactamente la misma forma `name-N` que el valor predeterminado de `SessionId`** (`` `session-${++counter}` `` en `packages/core/session/src/index.ts`). Un id de trabajo de bash y un id de sesión son intercambiables trivialmente en un punto de llamada y el compilador no dice nada. Es un id de cara al modelo (el modelo devuelve `job_id` a `bash_output`/`bash_kill`), de modo que una confusión aquí es alcanzable desde entrada no confiable.

El **token de propietario** de bash es el subcaso relacionado: `ShellExecRequest.owner?: string` y `ShellExecSpec.owner: string | undefined` (`packages/shell/shell/src/types.ts`) están documentados como una clave de aislamiento deliberadamente opaca, pero en cada llamante en vivo el valor ES el `Agent.id`/`SessionId` compartido del agente propietario (`callerToken = (exec) => exec.agent?.id` en `packages/shell/tool-bash/src/index.ts`) con un nombre distinto local al seam. Se compara para el control de acceso (`owner !== callerToken(exec)`), de modo que un string bien tipado pero no coincidente aquí es un bug de aislamiento entre sesiones que el sistema de tipos hoy no puede detectar. Este es el alias del id compartido cubierto por la [decisión de identidad unificada de agent/sesión](../simplification/2026-06-20-unify-agent-and-session-id.es.md).

**Vacío 2 — erosión de la marca en las fronteras de los ids ya marcados.** Incluso `CallId` y `SessionId` degeneran de nuevo a `string` pelado exactamente en los lugares donde la confusión es más probable: los tipos de clave de registros/almacenes y los parámetros de métodos públicos. Los sitios representativos incluyen el almacén de sesiones, el registro de agentes (ambos con clave del `SessionId` compartido), los mapas de ids de llamada de presentación de herramientas, los registros de sesión de ACP y el coordinador de persistencia. Una marca que se pierde en una clave de colección no aporta nada en las búsquedas: el valor de las marcas existentes queda en parte sin realizar.

## Decisión

Un cambio solo de tipos. Las marcas son conversiones de costo cero; nada del comportamiento en tiempo de ejecución, la serialización, la comparación o el formato de red cambia. La decisión tiene tres partes, todas ellas respetuosas con la política existente de «no todo string».

- **Marca el id de trabajo de bash.** Añade `BashTaskId = Branded<'BashTaskId'>` y su fábrica homónima en `packages/shell/shell/src/types.ts` (el paquete que posee el id), importando `Branded` de `@deepseek-ai/dsh-brand` exactamente como hace `SessionId`. La primitiva de marca vive en el paquete de utilidades `dsh-brand`, libre de dependencias, precisamente para que `dsh-shell` pueda marcar sus ids dependiendo solo de él — nunca arrastra `dsh-llm` (ni `dsh-session`) solo para llegar a `Branded`. Enhebra la marca por `BashTask.id`, los métodos de la Service Definition `ShellExecutor` (`get`/`ownerOf`/`readOutput`/`kill`), el sitio de generación en `dsh-bash-local` (marca la salida del contador una sola vez, en la creación) y la superficie de validación/acceso de `dsh-tool-bash` (`validateJobId` devuelve un `BashTaskId`; `job_id` se marca en la frontera de la herramienta, donde llega el string del modelo).
- **Acuña una marca `OwnerToken` distinta.** Añade `OwnerToken = Branded<'OwnerToken'>` en `packages/shell/shell/src/types.ts`; tipa `ShellExecRequest.owner` / `ShellExecSpec.owner` / `ShellExecutor.ownerOf` como `OwnerToken | undefined`. El Consumer `dsh-tool-bash` convierte el id compartido del agente (`SessionId`) en un `OwnerToken` en la frontera — el único lugar donde se encuentran los dos vocabularios. La Service Definition de bash nunca importa `dsh-session`. (La justificación está en la siguiente sección.)
- **Detén la erosión de la marca.** Propaga las marcas existentes a los tipos de clave de `Map` y a los parámetros de métodos públicos enumerados en el Vacío 2 — `Map<SessionId, Session>`, `Map<SessionId, Agent>`, `get(id: SessionId)`, `Map<CallId, …>`, la superficie `SessionId` de ACP y el `Map<SessionId, …>` del coordinador. Esta es la mayor parte mecánica del cambio y la que hace que las marcas existentes soporten de verdad las búsquedas, no solo los campos de struct.

Forma ilustrativa (el patrón de fábrica es idéntico al de las tres marcas existentes):

```ts ignore-check
import type { Branded } from '@deepseek-ai/dsh-brand'

/** A background bash task handle (generated `bash-N` by the local executor). */
export type BashTaskId = Branded<'BashTaskId'>
export function BashTaskId(id: string): BashTaskId {
  return id as BashTaskId
}

/** A bash task's opaque isolation key — the consumer's owner identity, NOT the bash seam's. */
export type OwnerToken = Branded<'OwnerToken'>
export function OwnerToken(id: string): OwnerToken {
  return id as OwnerToken
}
```

## Alternativas consideradas

### ¿Por qué no tipar `owner` como `SessionId`?

El atajo obvio es tipar `owner` directamente como `SessionId` — siempre lo es. Lo rechazamos. El seam del executor de bash es un seam de capacidad (Service Definition `dsh-shell`, Service Provider `dsh-bash-local`, Consumer `dsh-tool-bash`) y su token de propietario está *documentado como deliberadamente opaco*: el executor «nunca lo interpreta (no hay política de acceso que viva en el seam — eso es trabajo del Consumer)» (`packages/shell/shell/src/types.ts`). Tipar el campo de la Service Definition como `SessionId` importaría el vocabulario de `dsh-session` en un paquete que no debe saber qué significa un token de propietario — acoplaría un backend de ejecución genérico al modelo de sesión y contradeciría el diseño de token opaco. Un executor remoto o en sandbox que sustituya a `dsh-bash-local` no debería heredar una dependencia de sesión. La marca `OwnerToken` distinta mantiene el seam desacoplado: `dsh-shell` solo sabe que «un propietario es algún token opaco con marca», y el Consumer `dsh-tool-bash` — que ya decide la política de acceso — es la única frontera que convierte su `SessionId` en un `OwnerToken`. La marca sigue aportando la ganancia de seguridad (no puedes pasar un `BashTaskId` ni un string crudo donde se espera un propietario) sin el acoplamiento.

## Fuera de alcance / extensiones posibles

Se mantiene deliberadamente estrecho según la política de «no todo string necesita una marca». Cada uno de estos es una marca futura plausible, aplazada con una razón, no un compromiso:

- **`ModelId`** (`GenerateOptions.model`, la clave del registro de adaptadores de `LlmRuntime`) — una clave de búsqueda real entre paquetes (config → agent → llm → adapter); una marca razonable como siguiente paso, excluida solo para mantener centrado el radio de impacto de esta decisión.
- **`ToolName`** (la clave de `ToolRuntime`) — definido por el autor, legible para humanos y rara vez confundible con otro id; el candidato más débil, probablemente no merece una marca.
- **`ErrorCode`** (`HarnessError.code`) — un vocabulario cerrado (`ABORTED`, `NO_ADAPTER`, …), no un id por instancia; mejor servido por una unión de literales de string que por una marca, en todo caso.
- **Ordinales numéricos** — el número de turno, el número de paso y el `seq` de evento son `number`, no `string`, así que `Branded<string>` no aplica; una variante paralela `number & { readonly [BRAND]: B }` podría marcarlos, pero son ordinales posicionales que rara vez se pasan entre fronteras, así que el beneficio es bajo.
- **Construcción validada** — las fábricas de marca son conversiones puras sin comprobación en tiempo de ejecución, y cada frontera (`sessionId` de ACP, `call.id` emitido por el provider, el respaldo de string vacío en `dsh-llm-deepseek`) confía hoy en el string crudo. Un acompañante `SessionId.parse()` / `isValid()` que lance una excepción con entrada malformada en las fronteras es un vacío real, pero es un cambio de *comportamiento en tiempo de ejecución* con su propio diseño (¿qué es «malformado»? ¿qué ocurre en caso de fallo?) y pertenece a su propia decisión, no empaquetada en este cambio solo de tipos.

## Verificación

Los invariantes asentados: `BashTaskId` y `OwnerToken` están definidos en `dsh-shell` y enhebrados de extremo a extremo (Service Definition, el sitio de generación de `dsh-bash-local`, la herramienta de cara al modelo de `dsh-tool-bash`) sin dependencia de `dsh-shell` respecto a `dsh-session`; ninguna colección con clave de un id marcado en alcance (`CallId`/`SessionId`/`BashTaskId`) usa como clave un `string` pelado; los parámetros de métodos públicos y las firmas exportadas conservan la marca; y las marcas se construyen mediante la fábrica de conversión en cada frontera donde entra un string crudo (id de llamada del provider, id de sesión de ACP, `job_id` suministrado por el modelo), nunca con `as` dispersos.

## Consecuencias

- **Cambio mecánico en dos superficies.** Propagar las marcas toca el seam de bash (Service Definition + Service Provider + Consumer), la superficie de id de sesión de ACP y el coordinador de persistencia. El cambio es amplio pero de baja gravedad: un punto omitido es un error de compilación, no un bug silencioso. El cambio es observablemente solo de tipos — sin diferencia de comportamiento en instantáneas ni e2e. Se sitúa junto a la [decisión de identidad unificada de agent/sesión](../simplification/2026-06-20-unify-agent-and-session-id.es.md) porque ambas tocan la frontera de id de sesión / token de propietario; `OwnerToken` sigue siendo distinto del id unificado por la razón de desacoplamiento anterior.
- **Las marcas no validan.** Una marca es una salvaguarda contra la confusión, no una prueba de corrección: un id de sesión erróneo que sigue siendo un string bien formado pasa el comprobador de tipos exactamente como antes. Esta decisión no cierra ese vacío (ver Fuera de alcance) — solo detiene el error de categoría de pasar el tipo equivocado de id.
- **La línea de «dónde detenerse» sigue siendo una decisión de criterio.** Marcar `BashTaskId` pero no `ToolName`, `OwnerToken` pero no `ModelId`, es una decisión de gusto sobre qué strings «podrían confundirse de forma plausible». Los revisores razonables pueden querer más o menos; la política en `brand.ts` es el desempate, y esta decisión se inclina por los ids que están de cara al modelo o se usan para el control de acceso.
