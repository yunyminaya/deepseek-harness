# @deepseek-ai/dsh-storage-domain

[English](README.md) | Español

Forma de datos de dominio para el hub de almacenamiento de DeepSeek Harness: expone el servicio inyectable `ctx.storageDomain` y la proyección `ctx.storage.domain` correspondiente después de que se registre cada backend configurado. Un dominio se declara una vez con `defineDomain` (schemas de registro zod, tipos derivados con `z.infer`), se abre con `DomainFacility.open` y se sirve desde un estado autoritativo en memoria — las lecturas son síncronas, las escrituras se serializan en una cadena por dominio, alcanzan la durabilidad primero en el backend enrutado y luego actualizan la memoria y emiten `domain/changed`. El consumidor que abre posee el ciclo de vida del handle y lo libera con `Domain.close()` (idempotente; típicamente su propio disposer de `ctx.effect`); los dominios que siguen abiertos cuando el plugin se desmonta los cierra la facility.

El fundamento de diseño, la semántica de apertura y la división de capas storage/domain viven en la [Agent Note](../../../.agents/notes/proposed/architecture/2026-07-24-domain-kv-storage-and-workspace.es.md).

## Configuración

| clave | significado |
|---|---|
| `backend` | Nombre del backend por defecto para cada dominio (obligatorio; no existe un medio universalmente correcto). |
| `routes` | Sobrescrituras por dominio: nombre de dominio → nombre de backend. |

## Experiencia del modelo

### Estado de dominio durable

#### Qué ve el modelo

Nada. El paquete no registra herramientas, no inyecta prompts y no añade eventos de sesión; almacena datos no-sesión (registros de workspace, futuros sidecars de sesión) detrás de `ctx.storageDomain` y emite solo el evento `domain/changed` en proceso, que llega a un modelo solo si un paquete Consumer lo renderiza a través de su propia superficie documentada.

#### Efecto de tokens

Cero. Ningún texto de este paquete entra en una solicitud de modelo.

#### Efecto de KV Cache

Independiente: las lecturas y escrituras de dominio nunca tocan prefijos de solicitud, así que nada aquí puede invalidar la reutilización de la caché del provider.

## Limitaciones conocidas y trabajo diferido

- **Visibilidad de cambios de un solo proceso** — `domain/changed` es un evento en proceso; un segundo proceso host o una GUI que se reconecta no observa cambios hasta que aterrice el patrón de revisión entre procesos diferido en la Agent Note.
- **Sin transacciones entre tablas, índices secundarios ni claves de múltiples segmentos** — cada escritura toca un registro; los triggers y puntos de rework para estas extensiones quedan aplazados en la lista de trabajo diferido de la Agent Note.
