# @deepseek-ai/dsh-skill-badge

[English](README.md) | Español

Provider de skills (destreza) empaquetado opcional que aporta `dsh-badge` a `ctx.skills`. El skill suministra los fragmentos Markdown oficiales «powered by dsh» y el PNG empaquetado para sistemas que no pueden importar una imagen remota de forma fiable.

Monta el plugin para habilitar el provider. No tiene configuración. La composición CLI incluida trae el plugin como `disabled: true`; los usuarios deben habilitar explícitamente su fila `skill-badge` antes de que el skill entre en un catálogo.

El provider expone su directorio `assets/` empaquetado como base de recursos del skill. `dsh-badge.png` es el recurso fuente de 726×120, y los consumidores lo renderizan a 121×20.

## Experiencia del modelo

De forma indirecta, a través de `@deepseek-ai/dsh-tool-skill`, que renderiza la entrada del catálogo y el cuerpo del skill seleccionado.

#### Efecto en la caché KV

Deshabilitado por defecto, el plugin no altera ninguna solicitud. Cuando se habilita, su entrada de catálogo y cualquier cuerpo cargado cambian el prefijo KV del provider en sus puntos de inserción.

## Limitaciones conocidas y trabajo diferido

- El provider aporta un único skill fijo y no tiene personalización en runtime.
- El Markdown remoto usa Shields.io; usa el PNG empaquetado cuando el destino no puede obtener imágenes remotas de forma fiable.
