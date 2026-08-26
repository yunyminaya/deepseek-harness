# @deepseek-ai/dsh-spill

[English](README.md) | Español

El **`SpillStore`** (`ctx.spillStore`) define QUÉ hace un backend de spill — persistir el texto sobredimensionado de una herramienta y devolver un localizador visible para el modelo más una guía de recuperación — sin decir CÓMO.

Este paquete es un tercio de la capacidad de spill, dividido para que cada preocupación evolucione (e intercambie) de forma independiente:

| Paquete | Rol |
|---|---|
| `@deepseek-ai/dsh-spill` (este) | Service Definition: servicio abstracto + tipos de vocabulario |
| `@deepseek-ai/dsh-spill-local` | Service Provider: archivos privados con alcance de sesión en el sistema de archivos del host |
| `@deepseek-ai/dsh-spill-policy` | Consumer: la política de resultados de herramienta que hace spill de los resultados finales sobredimensionados |

La división refleja los seams de shell/fs. Un backend remoto o virtual futuro (p. ej. un URI `spill://…`, una clave de base de datos o una herramienta de recuperación específica del backend) implementa esta Service Definition sin tocar el plugin de la política.

## API del servicio (`ctx.spillStore`)

| Miembro | Semántica |
|---|---|
| `saveText(input)` | Persiste `input.content` tal cual; resuelve con un `SpillRef` (localizador opaco, bytes exactos escritos y pista de recuperación). **Rechaza ante un fallo real de almacenamiento** (permisos, ENOSPC, backend no disponible) — el llamador decide cómo degradar. |

El almacenamiento se agrupa por la sesión `owner` de la solicitud como espacio de nombres del momento del guardado; el backend elige su propia representación privada y puede derivar nombres de — nunca confiar como ruta en — el `suggestedName` del llamador. El seam posee solo el almacenamiento: SIN política de retención (eso es de [`@deepseek-ai/dsh-output-retention`](../../util/output-retention)), SIN sustitución de resultados de herramienta (eso es de `@deepseek-ai/dsh-spill-policy`), SIN API de recuperación/búsqueda (el `retrievalHint` del backend le dice al modelo qué hacer con el localizador).

## Vocabulario

`SaveTextSpill` (owner, source, suggestedName, content) es la solicitud; `SpillRef` (locator, bytes, retrievalHint) es el resultado. `SpillLocator` es un tipo [branded](../../util/brand) y se renderiza al modelo como una cadena opaca — una ruta local para `dsh-spill-local`, pero un backend futuro puede devolver un URI, una clave o un token de comando sin cambiar los consumers de políticas/herramientas. `SpillOwner.sessionId` es el espacio de nombres de almacenamiento del momento del guardado: las sesiones bifurcadas heredan los localizadores existentes del log sembrado sin copiarlos ni reasignarlos, y los nuevos spills posteriores a la bifurcación usan el id de la sesión hija. `SpillSource` registra el `toolName`, `callId` y `label` productores para el nombrado e inspección del backend, no para el control de acceso. Consulta `src/types.ts` para los contratos completos.

Consulta la [Agent Note de spill de salida de herramientas](../../../.agents/notes/implemented/architecture/2026-07-08-tool-output-spill-files.es.md) para el fundamento de diseño, incluido por qué la creación pertenece al seam de spill del runtime en lugar de a la herramienta `write` visible para el modelo.

## Experiencia del modelo

De forma indirecta, a través de los consumidores de spill que renderizan un localizador del backend y una guía de recuperación.

#### Efecto de KV Cache

Sin invalidación directa; el consumidor nombrado es dueño de cualquier cambio de prefijo de solicitud.

## Limitaciones conocidas y trabajo diferido

- **El seam no tiene API de recuperación ni de borrado** — los consumidores solo pueden renderizar el localizador y la guía del backend; la semántica de ciclo de vida y de acceso sigue siendo específica del backend.
- **El almacenamiento no es control de acceso** — `SpillOwner` pone los escritos en espacios de nombres pero no autoriza lecturas de un localizador; cada backend y cada consumer de recuperación deben imponer su propia frontera.
