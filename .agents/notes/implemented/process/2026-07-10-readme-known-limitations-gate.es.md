# Agent Note: Una sección de Limitaciones Conocidas controlada por una puerta en cada README de paquete

Status: implemented

[English](2026-07-10-readme-known-limitations-gate.md) | Español

## Problema

El [estándar de documentación](../../../../docs/AGENTS.md) asigna las limitaciones a los READMEs de paquete. Sin una forma compartida, una sección omitida no distingue una ausencia auditada de documentación olvidada, y los encabezados variantes impiden una búsqueda a nivel de repositorio.

## Decisión

Cada manifest de paquete bajo `packages/<group>/<pkg>/package.json` tiene un README hermano con la sección canónica `## Known Limitations and Deferred Work`. Sus viñetas registran huecos duraderos para el consumidor y restricciones no obvias para el mantenedor de las que ese paquete es dueño; la limpieza ordinaria permanece en su TODO del código fuente o en el Agent Note propietario. La [`verify-package-readme-limitations` gate](../../../../scripts/verify-package-readme-limitations.ts) deriva el conjunto de paquetes desde los manifests, rechaza READMEs ausentes y exige exactamente un h2 canónico con al menos una viñeta de nivel superior. Encabezados cuasi idénticos como «Limitations», «Deferred», «What is NOT here» o «Non-goals» fallan.

Un paquete sin nada que declarar se lista en `NO_LIMITATIONS` y omite la sección. Añadir una limitación exige eliminar la entrada; los renombres y las eliminaciones fallan porque cada entrada debe nombrar un paquete escaneado.

La puerta comprueba presencia, forma y la allowlist. La revisión bajo los estándares de documentación y [prosa](../../../skills/dsh-prose-standard/SKILL.md) es dueña de la cobertura y la precisión. La regla permanente vive en [packages/AGENTS.md](../../../../packages/AGENTS.md).

## Alternativas consideradas

- **Encabezados libres** — no se pueden buscar uniformemente y siguen necesitando detección de cuasi idénticos.
- **Exigir una sección vacía o «None.»** — el texto de relleno puede permanecer después de que un paquete gane una limitación; una allowlist hace explícita y revisable la ausencia.
- **Imponer un techo de palabras** — los recuentos legítimos de limitaciones varían, así que la revisión gobierna este nivel de README sin presupuesto.

## Consecuencias

- Los paquetes nuevos declaran limitaciones que califican o se unen explícitamente a la allowlist; las secciones ausentes, derivadas y vacías fallan `doc-sync` localmente y en CI.
- La puerta añade un script TypeScript sin dependencias a `doc-sync`.
- Renombrar el encabezado aplicado exige cambiar juntos el script y cada README de paquete.
