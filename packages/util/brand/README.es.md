# dsh-brand

[English](README.md) | Español

La primitiva de tipado nominal `Branded<B>` — un paquete diminuto, **solo de tipos** (sin código de runtime, sin dependencia de paquetes del harness) compartido por todos los paquetes que son dueños de un id que cruza fronteras.

## Qué es `Branded`

Un brand hace no intercambiables a nivel de tipos las cadenas estructuralmente idénticas: un `SessionId` no puede pasarse donde se espera un `CallId`, aunque ambos sean `string`s planos en runtime.

```ts
import type { Branded } from '@deepseek-ai/dsh-brand'

export type SessionId = Branded<'SessionId'>

/** Brand a string as a SessionId (a plain cast — zero runtime cost). */
export function SessionId(id: string): SessionId {
  return id as SessionId
}
```

La construcción pasa por la fábrica por-id del paquete propietario. La comparación, el registro, la serialización JSON y el formato de wire se comportan como los de una cadena ordinaria; el brand se borra en tiempo de compilación.

## Política: marca los ids que cruzan fronteras de paquete

Un paquete marca los ids de los que es dueño — `CallId` en `dsh-llm`, el `SessionId` compartido de agent/session en `dsh-session` y `JobId` en `dsh-jobs`. Marca los ids entre paquetes que podrían confundirse de forma plausible; no toda cadena necesita uno.

Este paquete es dueño solo de la primitiva. Mantenerlo sin dependencias permite que `dsh-jobs`, por ejemplo, marque `JobId` sin importar un paquete de capacidad no relacionado solo para llegar a `Branded`.
