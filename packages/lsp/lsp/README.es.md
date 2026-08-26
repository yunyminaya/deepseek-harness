# @deepseek-ai/dsh-lsp

[English](README.md) | Español

El **seam de capacidad LSP**: un `LspService` abstracto (`ctx.lsp`) que define QUÉ navegación semántica de código tiene el harness — ir a la definición, buscar referencias, buscar implementaciones, hover — sobre providers de servidor de lenguaje, sin atar el contrato del modelo a subprocesos locales.

Este paquete es dueño del rol de Service Definition de la capacidad LSP:

| Paquete | Rol |
|---|---|
| `@deepseek-ai/dsh-lsp` (este) | Service Definition: el servicio, el registro de providers indexado por id con marca + mapeo de extensiones, la selección por consulta, el vocabulario de petición/resultado, la taxonomía `LspError` |
| `@deepseek-ai/dsh-lsp-stdio` | Service Provider: un backend local genérico que registra providers de servidor de lenguaje stdio configurados |
| `@deepseek-ai/dsh-tool-lsp` | Consumer: la herramienta `lsp` orientada al modelo sobre `ctx.lsp` |

El seam expone exactamente cuatro operaciones semánticas — `goToDefinition`, `findReferences`, `goToImplementation`, `hover` — y ninguna salida de escape JSON-RPC genérica, de modo que ningún payload de protocolo ni comando/mutación sin revisar llega a un provider a través de `ctx.lsp`.

## API de servicio (`ctx.lsp`)

| Miembro | Semántica |
|---|---|
| `registerProvider(provider)` | Registra un backend, reservando atómicamente su `id` con marca y cada extensión de archivo normalizada. Cualquier entrada inválida o conflicto no publica nada y lanza `LspError` (`LSP_INVALID_PROVIDER` / `LSP_CONFLICT`). Devuelve un disposer que libera todas las reservas. Se libera con la fiber que hace la llamada. |
| `query(request, signal?)` | Selecciona el provider por la extensión final del archivo, deriva el `languageId` del mapeo de ese provider y ejecuta una consulta. Sin coincidencia, lanza `LspError` `LSP_UNAVAILABLE`. |

La selección es por consulta e independiente del orden: un provider posee un conjunto de extensiones en exclusiva, así que el orden de registro y de HMR nunca cambia el enrutado. Las claves de extensión se normalizan a minúsculas con punto inicial; el `languageId` solo sincroniza el documento transitorio, nunca participa en la selección. La primera versión no tiene selector de glob, de id de lenguaje ni de ruta explícita.

Los providers registran **capacidades**, no herramientas. `dsh-tool-lsp` es el único dueño del nombre, la descripción, la guía de prompt, el schema y la presentación orientados al modelo.

## Vocabulario

`LspQueryRequest` (`operation`, `filePath`, `position`, `workspaceRoot`) — todos los campos obligatorios, de modo que ningún campo necesita valores por defecto de implementación y no hay paso de `resolve()`. Las posiciones y rangos son UTF-16 de base cero, acordes al protocolo; la herramienta es dueña de la convención de cursor de base uno. `findReferences` siempre incluye las declaraciones — los providers lo imponen internamente, así que los llamadores no reciben ninguna bandera. `LspQueryResult` es una unión discriminada CERRADA: `{ kind: 'locations'; locations; resolvedWorkspaceUri }` para navegación, `{ kind: 'hover'; hover }` para hover (contenido o `null`) — los Consumers hacen `switch` para agotar los casos, de modo que un nuevo brazo rompe la compilación hasta que se maneja. `resolvedWorkspaceUri` es la URI `file:` canónica del workspace del provider; los llamadores relativizan las URIs de ubicación contra ella en lugar de aplicar reglas de ruta de la plataforma host a la raíz de petición posiblemente con symlinks. Ver `src/types.ts` para los contratos completos y `src/index.ts` para los códigos `LspError`, incluidos `LSP_DISPOSED` y `LSP_MALFORMED_RESPONSE`.

## Experiencia de modelo

Indirectamente, a través de `dsh-tool-lsp`, que es dueño del schema `lsp` orientado al modelo, del prompt y de los resultados renderizados, mientras que este registro no aporta prompt ni schema por sí mismo.

#### Efecto de KV Cache

Sin invalidación directa; `dsh-tool-lsp` es dueño de los cambios de prefijo de petición.

## Limitaciones conocidas y trabajo diferido

- **Propiedad exclusiva de extensiones dentro de un runtime** — dos providers no pueden reclamar ambos `.ts`, ni siquiera con ids de lenguaje distintos; las superposiciones hacen fallar el registro. La extensión prevista es un selector configurado por despliegue por encima de los registros, que puede relajar la reserva exclusiva sin añadir elección de provider a la entrada del modelo ([Agent Note del seam](../../../.agents/notes/implemented/architecture/2026-07-15-lsp-capability-seam.es.md)).
- **Solo cuatro operaciones** — los símbolos y la jerarquía de llamadas están diferidos (necesitan schemas distintos); los diagnósticos necesitan reglas separadas de frescura/acumulación; las mutaciones (rename, code actions, formato) requieren herramientas separadas con integración de vista previa, permiso y política de escritura.
- **Sin API de observación** — la disponibilidad solo se observa ejecutando `query()` y enrutando los códigos `LspError` lanzados; no hay evento de cambio de provider ni consulta de estado de capacidad.
