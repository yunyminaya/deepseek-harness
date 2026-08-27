---
name: dsh-doc-site-sync
description: Úsalo al publicar, actualizar, mover o eliminar páginas del sitio de documentación de DeepSeek Harness; al editar los mapeos o la navegación de website/docs.ts; al diagnosticar una página que falta en el sitio VitePress; al corregir enlaces de documentación proyectada; o al ejecutar el flujo docs:dev, docs:check y doc-sync tras cambios en contenido del sitio web.
---

# Sincronizar el sitio de documentación de DeepSeek Harness

[English](SKILL.md) | Español

Mantén el Markdown del repositorio como la única fuente de contenido editable.
Trata el sitio web como una proyección probada: [website/docs.ts](../../../website/docs.ts) selecciona las páginas públicas, [scripts/project-doc-site.ts](../../../scripts/project-doc-site.ts) las reescribe al árbol desechable `website/.generated/`, y VitePress construye ese árbol.
El build además emite un gemelo raw-Markdown de cada ruta (URL de la página sin la barra final, más `.md`; las rutas índice también reciben un alias de nivel padre) y un índice raíz `llms.txt`; ambos derivan del mismo manifest y projector, así que publicar, mover o eliminar una página los actualiza automáticamente y `docs:build` falla cuando falta uno.

Las traducciones del repositorio siguen el contrato de emparejamiento entre hermanos: el inglés `foo.md`, el chino `foo.zh.md` y `foo.i18n.yaml` viven juntos.
Nunca crees directorios `zh-CN/` u otros directorios de locale para el contenido del sitio.
Los árboles de rutas del sitio son independientes de ese layout fuente: `foo.zh.md` se proyecta a la ruta raíz y `foo.md` se proyecta a la ruta correspondiente `/en/`.

## Lee los contratos propietarios

- Lee [docs/AGENTS.md](../../../docs/AGENTS.es.md) y usa [dsh-doc-standards](../dsh-doc-standards/SKILL.es.md) al decidir dónde pertenece el contenido o al cambiar prosa de documentación del producto.
- Para una fuente bilingüe editada, sigue la ruta rutinaria ligera en [docs/AGENTS.md](../../../docs/AGENTS.es.md#writing-rules) y el [contrato de emparejamiento](../../../docs/i18n/README.es.md); nunca invoques automáticamente el skill de traducción extendida.
- Lee el tipo actual `DocsPage` y las entradas en [website/docs.ts](../../../website/docs.ts) antes de cambiar el manifest; no dependas de un conjunto de campos recordado.
- Lee [website/.vitepress/config.ts](../../../website/.vitepress/config.ts) antes de añadir una sección nueva, una colección de sidebar, un locale o un elemento de navegación de nivel superior.

## Clasifica el cambio

- **Editar una página ya publicada:** cambia solo su fuente Markdown canónica.
No toques el manifest salvo que cambie su ruta o su metadata de navegación.
- **Publicar una página nueva:** créala en su tier propietario bajo `docs/` y luego añade una entrada al manifest.
- **Renombrar, mover o eliminar una página:** actualiza atómicamente el archivo canónico, la entrada del manifest y los enlaces entrantes del repositorio.
Elimina entradas obsoletas del manifest; `docs:check` rechaza fuentes ausentes.
- **Publicar un catálogo generado:** mapea el archivo generado bajo `docs/`, pero cambia su generador o la metadata de su fuente en vez de editar el catálogo a mano.
- **Cambiar la estructura del sitio:** actualiza el manifest para páginas ordinarias; actualiza la configuración de VitePress solo cuando el modelo existente de sidebar, sección o locale no pueda expresar el cambio.

Nunca edites ni commitees `website/.generated/`, `website/.cache/` ni `website/.dist/`.
Salvo `website/AGENTS.md`, nunca añadas Markdown bajo `website/`; los directorios de locale y rutas como `website/zh-CN/`, `website/en/` y `website/api/` son layouts fuente no válidos.
Mantén los catálogos generados bajo `docs/`, protégelos allí con freshness gates y publícalos a través del manifest.

## Añade o actualiza una entrada del manifest

Define deliberadamente cada campo de `DocsPage`:

- `source`: ruta del Markdown canónico relativa al repositorio.
Para un par bilingüe completo, añade la ruta inglesa `.md` mediante `pairedPages()`; así deriva el `.zh.md` hermano, los locales de contenido y los aliases equivalentes.
- `route`: ruta pública de VitePress incluyendo el sufijo `.md`.
- `label`: etiqueta del sidebar, no necesariamente el H1 del documento.
- `sidebar`: reutiliza `zh-guide`, `zh-develop` o `en-docs` salvo que la arquitectura de información realmente necesite otra colección.
- `section`: reutiliza una sección existente cuando sea posible.
Si añades una, colócala también en `sectionOrder` dentro de la configuración de VitePress.
- `order`: orden estable dentro de la sección.
- `sourceAliases`: rutas adicionales opcionales del repositorio que deben resolver a esta página cuando se proyecten enlaces.
No crea otra ruta pública.

Usa `mirroredPages()` solo para una fuente que deliberadamente deba caer en el mismo idioma disponible en ambos árboles de rutas.
Convierte esa entrada a `pairedPages()` cuando se añada su contraparte.
Mantén el manifest como una allowlist pública explícita.
No publiques RFCs, post-mortems, guías de testing, `AGENTS.md` ni flujos de trabajo de mantenimiento solo porque existan bajo `docs/`; añade material interno solo cuando el usuario amplíe explícitamente lo que publica el sitio.

## Conserva el comportamiento de enlaces

Escribe enlaces Markdown normales relativos al repositorio en la documentación canónica.
El projector aplica estas reglas:

- Un destino presente en el manifest se convierte en una ruta relativa del sitio.
- Un destino existente fuera del manifest se convierte en un enlace al código fuente en GitHub, incluidos los sufijos de línea compatibles.
- Una imagen es la excepción: su archivo se copia al árbol generado y se referencia desde allí, así que el sitio la sirve independientemente de la visibilidad del repositorio.
Debe ser un archivo regular dentro del repositorio.
- Las URLs externas, las URLs absolutas del sitio, los enlaces de correo y los enlaces solo con fragmento permanecen sin cambios.
- Un destino relativo al repositorio que no exista hace fallar la proyección en vez de producir silenciosamente un enlace roto.
- Los fragmentos entre páginas usan el id de encabezado GitHub inglés como id canónico.
Si un encabezado redactado emite un id distinto en VitePress, coloca un `<a id="..."></a>` explícito inmediatamente antes; añade aliases generados en el generador propietario.

No escribas rutas específicas del sitio web en el Markdown canónico solo para satisfacer a VitePress.
Usa `sourceAliases` para enlaces estilo directorio del repositorio que deban resolver a una página índice mapeada.

## Previsualiza y valida

Ejecuta la vista previa local mientras editas:

```sh
pnpm docs:dev
```

El servidor de desarrollo observa los archivos fuente mapeados y los reproyecta.
Reinícialo tras cambiar el manifest si la fuente nueva no se detecta automáticamente.

Ejecuta el gate web enfocado antes de dar el mapping por válido:

```sh
pnpm docs:check
```

Si las comprobaciones de enlaces Markdown pasan pero el build del sitio informa un fragmento ausente, sigue las rutas fuente y destino de `verify-doc-site-fragments`.
Conserva el id GitHub inglés con un alias explícito en el Markdown redactado o en el generador propietario.

Antes de commitear un cambio del sitio de documentación, ejecuta:

```sh
pnpm run doc-sync
pnpm run lint
git diff --check
```

Usa [dsh-pre-push-checks](../dsh-pre-push-checks/SKILL.es.md) antes de hacer push.
Informa los archivos canónicos cambiados, las entradas del manifest añadidas o eliminadas, las rutas públicas afectadas y las comprobaciones exactas ejecutadas.

## Mantén separado el despliegue

Sincronizar contenido en el build de VitePress no lo publica en Internet.
No añadas permisos de GitHub Pages, flujos de despliegue, dominios personalizados ni hosting público salvo que el usuario pida explícitamente el despliegue y confirme la política de hosting.
