# Agent Note: Markdown cross-link validity linting

Status: implemented

English | [中文](2026-06-18-markdown-cross-link-lint.zh.md) | Español

## Problema

La documentación de este repo se enlaza entre sí por ruta relativa — `[topic](../implemented/2026-…-….md)`, `[the cookbook](adding-a-tool.md)`, `[architecture.md](../../architecture.md)`. Nada verificaba que esos destinos existieran. Un renombrado o un movimiento rompe silenciosamente todos los enlaces entrantes, y la rotura es invisible hasta que un lector pulsa uno. La [aplicación de doc-sync](../../archived/process/2026-06-11-doc-sync-enforcement.md) ya había mecanizado dos clases de deriva documental (bloques de código no compilables y una tabla de taxonomía de eventos desfasada) y [verify-md-wrap](../../archived/process/2026-06-11-doc-sync-enforcement.md) una tercera (prosa con saltos de línea forzados) — pero un enlace cruzado muerto es una cuarta clase igual de mecánica que todavía se verificaba a ojo.

El caso detonante fue la reorganización del árbol de Agent Notes que introdujo esta puerta: unificar `docs/adr/` + `.agents/notes/` en un único `.agents/notes/` con subcarpetas `proposed/`/`implemented/`/`rejected/` renombró a mano unos cuarenta enlaces entre documentos. Un solo camino mal tecleado habría enviado un enlace roto sin nada que lo detectara.

## Decisión

Una cuarta puerta de `doc-sync`, `verify-md-links` (`scripts/verify-md-links.ts`), que refleja el estilo de `verify-md-wrap` (tsx ESM, basado en AST, verificar-no-generar):

- Interpretar cada archivo Markdown en el ámbito con `mdast-util-from-markdown` + GFM y recorrer todos los nodos `link`, `image` y `definition`.
- Comprobar un destino solo cuando es una **ruta relativa**. Omitir las URLs con esquema (`https:`, `mailto:`, …), las relativas al protocolo (`//host`), las absolutas desde la raíz (`/path` — no hay base estable en un checkout) y las anclas puras dentro de la página (`#section`). Quitar cualquier `#fragment`/`?query`, resolver la ruta contra el directorio del archivo que enlaza y afirmar que existe en el disco.
- Reportar y nunca reescribir; salir con código distinto de cero al primer enlace roto encontrado.

El ámbito coincide con el de las demás puertas más la pareja de AGENTS.md y el Markdown de skills de agente redactado en el repo bajo `.agents/skills/` (esos archivos de skill enlazan hacia el árbol de documentación, así que esta reorganización también reescribió enlaces en ellos): `README.md`, `docs/**/*.md`, `packages/*/README.md`, `AGENTS.md`, `packages/AGENTS.md`, `.agents/skills/**/*.md`, deduplicados por ruta real (los symlinks de `CLAUDE.md` resuelven sobre los archivos AGENTS.md). Está cableada en `doc-sync`, de modo que los cambios de documentación pertinentes y el CI ejercitan la misma comprobación de enlaces rotos.

La puerta ahora comprueba también las anclas `#fragment` de los destinos Markdown — incluidas las del mismo archivo — contra los slugs de los encabezados y los `<a id>` explícitos; la [decisión de anclas de fragmento](2026-08-09-md-fragment-anchor-gate.es.md) es dueña de ese mecanismo y de las reglas de slug.

## Alternativas consideradas

**Comprobación de validez a nivel de ancla** — aplazada aquí por más pesada y de menor valor (los enlaces muertos a nivel de archivo eran el fallo que realmente había mordido), dejando que los autores verificaran ellos mismos las anclas `#fragment`. Esa regla manual no se sostuvo; la [decisión de anclas de fragmento](2026-08-09-md-fragment-anchor-gate.es.md) añadió después la comprobación.

## Consecuencias

- Los renombrados y movimientos que huérfanan un enlace cruzado hacen fallar `doc-sync` y el CI en lugar de esperar a que un lector pulse un enlace muerto. Esto hizo que la reorganización de Agent Notes que introdujo la puerta fuese auto-verificable: la comprobación demuestra que ninguno de sus propios enlaces reescritos queda colgando.
- Un script tsx más, rápido, en la cadena de `doc-sync`; ninguna dependencia nueva (la pila mdast/GFM ya está en devDependencies para `verify-md-wrap`).
- La convención que esto impone — referenciar documentos entre sí mediante enlaces relativos comprobables por máquina, nunca mediante prosa suelta ni un número — está documentada en [docs/AGENTS.md](../../../../docs/AGENTS.es.md) para que los autores sepan que la puerta existe y por qué.
