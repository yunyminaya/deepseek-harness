# AGENTS.md — Ejemplos

[English](AGENTS.md) | Español

Composiciones de harness ejecutables. `examples/` es un miembro del workspace y la raíz de resolución de módulos para las configuraciones Cordis ejecutables y de prueba; no es un objetivo de build. [package.json](package.json) declara los paquetes que cargan esas configuraciones, mientras que el `package.json` privado de cada hoja sigue siendo solo metadatos.

Extrae la lógica reutilizable a `packages/`, donde se aplican las compuertas de cobertura por archivo y de README. Los ejemplos conservan solo el cableado de `cordis.yml`, los artefactos de demo y los escenarios e2e/de instantánea; los bins de los paquetes de aplicación son dueños del pegamento de arranque.

## Smokes e2e

Cada ejemplo tiene ambas:

- **Sin clave (keyless):** arranca el `cordis.yml` real a través del Loader, lo conduce y verifica la salida y una salida limpia. Detecta exportaciones inválidas del Loader que las pruebas montadas a mano pasan por alto ([post-mortem](../docs/postmortem/0001-acp-default-export-drops-inject.es.md)).
- **Con clave:** envía un prompt a un modelo real y verifica el estado externo, no lo que afirma el modelo. Se omite a sí misma sin `DEEPSEEK_API_KEY`; ver [testing.md](../docs/testing.md).

Los smokes de proceso sin clave usan `@deepseek-ai/dsh-loader-smoke` para la resolución del lanzamiento del Loader; las pruebas de terminal envuelven ese lanzamiento en una pseudo-terminal. Las pruebas aportan rutas, entorno, entrada y aserciones. Cada configuración Cordis de prueba confirmada vive en su hoja `examples/<agent>/` correspondiente. Asigna una configuración propiedad de un paquete a `examples/<agent>/tests/fixtures/<group>/<package>/cordis.yml`, mantén su driver y sus aserciones locales al paquete y declara cada paquete que nombre tanto en las referencias del `tsconfig.json` raíz como en `examples/package.json`.

No hagas inventario aquí de las pruebas de los ejemplos; los árboles `tests/` y los scripts de la raíz son la autoridad.

En `cordis.yml`, comenta solo el cableado no evidente, las consecuencias de orden de carga, el replay, los límites de seguridad y el alcance de la configuración. No narres las entradas visibles; usa [dsh-prose-standard](../.agents/skills/dsh-prose-standard/SKILL.md) para la cobertura requerida y el criterio editorial.

Ver [el AGENTS.md de la raíz](../AGENTS.md) para las convenciones de todo el repositorio y [docs/architecture.md](../docs/architecture.es.md) para el diseño.
