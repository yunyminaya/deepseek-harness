# @deepseek-ai/dsh-storage

[English](README.md) | Español

Hub de almacenamiento (`ctx.storage`) para datos no-sesión: un registro de backends con nombre más módulos de forma de datos montados. El hub no realiza IO por sí mismo — los backends poseen el medio y las formas de datos poseen la semántica. La [visión general de la familia de almacenamiento](../README.md.es.md) mapea esos paquetes; la [Agent Note de almacenamiento KV de dominio](../../../.agents/notes/proposed/architecture/2026-07-24-domain-kv-storage-and-workspace.es.md) registra el fundamento del diseño.

## Forma

- `ctx.storage.backend` — tabla nombre → backend. Varios backends permanecen montados lado a lado (`json`, `sqlite`); qué backend sirve a un consumidor es configuración de ese consumidor (la tabla de rutas de la capa de dominio), nunca una elección global del hub. `register()` devuelve el disposer; los nombres duplicados y las búsquedas desconocidas fallan ruidosamente.
- `ctx.storage.mount(form, facility)` / `ctx.storage.form(form)` — montaje de formas de datos. `StorageForms` es extensible mediante merge; la capa de dominio fusiona `domain` y se alcanza como `ctx.storage.domain`.
- Un backend posee un medio y expone las facetas de forma de datos que soporta. `kv` es la faceta actual; `src/backend.ts` es dueño de su contrato exacto.

## Experiencia del modelo

### Registros de backend y forma

#### Qué ve el modelo

Nada. `ctx.storage` es una tabla de registro del lado del host; el hub no registra herramientas, no inyecta prompts y no escribe eventos de sesión.

#### Efecto de tokens

Cero tokens directos en cada solicitud.

#### Efecto de KV Cache

Independiente de las solicitudes en vivo: el hub nunca toca un prefijo de solicitud, así que no puede invalidar la reutilización de la caché del provider.

## Limitaciones conocidas y trabajo diferido

- **`kv` es la única forma de datos** — los backends tienen hoy una sola faceta que implementar.
- **Las formas resuelven de forma perezosa** — leer `ctx.storage.domain` antes de que el plugin de dominio monte lanza `form-not-mounted`; las composiciones ordenan los plugins en consecuencia (la mala configuración falla ruidosamente en lugar de diferir en silencio).
