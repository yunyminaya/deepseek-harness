# Agent Note: Prune unused skill registry API

Status: rejected — el registro de skills directo en runtime es un camino de extensión intencional para plugins de terceros.

English | [中文](2026-07-12-prune-unused-skill-registry-api.zh.md) | Español

## Problema

El subsistema de runtime embebido del servicio de skills no tiene ningún llamador de producción de `ctx.skills.register()`. Este añade un nombre de provider `runtime` reservado, un mapa/rank/fuente de runtime, política de duplicados, una segunda revisión en las claves de caché, normalización, disposers y pruebas junto al contrato de provider que todas las skills enviadas ya usan. `SkillSummary.whenToUse` y el `path` de candidato/definición se interpretan y copian pero ningún consumidor de producción los lee: el catálogo del modelo renderiza nombre/descripción, la carga de recursos usa `resourceBase` y los providers son dueños de su locator. El punto de extensión `metadata`, deliberadamente abierto, se conserva.

## Propuesta

Eliminar `SkillRegistry.register()`, `SkillRegistration`, el pseudo-provider de runtime y las reglas de nombre reservado, las revisiones de runtime y las ramas de caché, y la normalización de fuente/rank exclusiva de runtime. Las pruebas que necesiten una skill embebida registran un provider real pequeño. Conservar `providerRevision` como la época de descubrimiento en curso, pero indexar los catálogos completados solo por cwd: cada mutación de provider limpia sincrónicamente la caché, y la comparación de revisión tras el await ya impide insertar trabajo caducado. Eliminar del contrato de skills y de las copias del provider local `whenToUse`, `SkillCandidate.path` y `SkillDefinition.path`, reteniendo las rutas de locator/raíz del provider; conservar `metadata`, `disableModelInvocation`, `source`, `provider`, `locator` y `resourceBase` ya sea como vocabulario de extensión deliberado o como campos consumidos por producción.

Modificar la Agent Note del sistema de skills, el README, el JSDoc, los catálogos y las pruebas. Las secciones de system-prompt de ámbito de agente, los providers de herramientas y las variables quedan explícitamente fuera de esta propuesta: el [contrato de contribuidores de ámbito de agente](../../implemented/architecture/2026-07-08-agent-scope-contexts.es.md) permite intencionadamente registrar los tres durante `setup(agentCtx)` a través del contexto propiedad del agente, de modo que la ausencia de un registro de ámbito fijo dentro del repo no es evidencia de no consumo.

## Alternativas consideradas

**Mantener el registro de skills en runtime para los embedders.** Es una comodidad intencional de definición directa y sincrónica en la Agent Note de skills ya implementada. Un pequeño wrapper de provider puede exponer los mismos datos embebidos bajo una vida útil propiedad de un effect, pero debe implementar `list()`/`get()` asíncronos, cargar con identidad de provider y aceptar la semántica de duplicados de provider. La propuesta elige ese único camino regular antes que preservar un segundo camino de ranking, validación, invalidación de caché y búsqueda.

## Criterios de aceptación

- La colección de skills tiene un único camino respaldado por providers, una clave de caché de completados solo por cwd y una época de revisión únicamente para la invalidación en curso; los campos de skill retenidos tienen un lector de producción o un contrato de extensión deliberado registrado.
- Las secciones de prompt de ámbito de agente, las variables, los providers de herramientas, las guardas de herramientas y el comportamiento de commit de salida estructurada en modo nativo y en Code Mode permanecen sin cambios.
- Pasan el typecheck, la cobertura, las instantáneas, el doc-sync, la verificación del module-graph, el build y la higiene.

## Riesgos

Esta es una contracción visible en compilación del registro de skills de prelanzamiento. Los consumidores programáticos externos de `list()`/`get()` pierden las pistas de enrutado `whenToUse` y el `path` de candidato/definición; el catálogo del modelo enviado nunca los renderiza y la resolución de recursos conserva su `resourceBase` explícito más el locator opaco propiedad del provider, pero esos campos no son observacionalmente idénticos. El análisis de frontmatter local de cada skill debe seguir preservando y validando el schema de metadatos soportado, y los providers externos siguen pudiendo suministrar fuentes de skills embebidas, de sistema de archivos, remotas o de otro tipo.
