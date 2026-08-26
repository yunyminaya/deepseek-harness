# AGENTS.md — Adaptador del sitio web de documentación

[English](AGENTS.md) | Español

Sigue las [instrucciones raíz](../AGENTS.es.md), el [estándar de documentación](../docs/AGENTS.es.md) y el [flujo de trabajo de sincronización del sitio de documentación](../.agents/skills/dsh-doc-site-sync/SKILL.es.md).

## Mantén el contenido de documentación fuera de este árbol

`website/` solo posee la configuración de VitePress, los assets de presentación y el manifest de publicación. Este archivo es el único archivo Markdown mantenido en este subárbol.

Mantén la prosa canónica y los catálogos generados en su nivel `docs/` propietario y expón luego las páginas seleccionadas mediante [docs.ts](docs.ts). Nunca añadas árboles de documentación con locale, ruta, API o copiados, como `website/zh-CN/`, `website/en/` o `website/api/`.

El proyector escribe Markdown desechable en el directorio ignorado `website/.generated/`. Nunca edites ni hagas commit de `.generated/`, `.cache/` o `.dist/`.

El build también emite el gemelo en Markdown crudo de cada ruta (con un alias de nivel padre por ruta de índice) y un índice raíz `llms.txt` dentro de `.dist/`, de modo que la URL de una página, menos cualquier barra final, más `.md`, la sirve como Markdown plano. Ambos se derivan del manifest de publicación en tiempo de build; ninguno es jamás un archivo de este árbol.

Ejecuta `pnpm docs:check` después de cambiar este subárbol; el gate rechaza cualquier Markdown adicional no ignorado bajo `website/`.
