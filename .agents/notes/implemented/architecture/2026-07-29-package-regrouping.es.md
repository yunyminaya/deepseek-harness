# Agent Note: Reagrupar packages/ según el agrupamiento medido

Status: implemented

[English](2026-07-29-package-regrouping.md) | Español

## Problema

La jerarquía de dos niveles `packages/<group>/<pkg>` ([decisión original](../../archived/architecture/2026-06-20-package-hierarchy.md)) se había desviado desde junio: 167 paquetes estaban en 42 grupos, y varias fronteras de grupo ya no coincidían con cómo se agrupan realmente los paquetes.

- `ui/` mezclaba cuatro planos sin relación: el canal de terminal humano (`tui`), la mitad de servidor JSON-RPC del SDK (`jsonrpc`, cuya dependencia peer de `dsh-sdk-protocol` lo ata a la pila de cableado del SDK), los seams de interacción humana (`user-questions`, `user-approval`, `permission`, `tool-ask-user`, `commands`) y el pegamento de boot neutro de canal (`app-boot`). Su propio README narraba la mezcla en lugar de enunciar un rol.
- La familia de sesión estaba fragmentada en cinco grupos — `session-persistence/`, `session-projection/`, `session-query/`, `session-title/` y `telemetry/` — aunque las aristas de dependencia medidas las unen (query → persistence, title → projection, projection → persistence; ver [docs/module-graph.md](../../../../docs/module-graph.es.md)).
- El grupo `timeout/` para una guarda de llamada de herramienta chocaba con `util/timeout`, la utilidad genérica de promesas.
- `cordis/` nombraba su grupo por el framework sobre el que se construye todo paquete, así que el nombre no discriminaba nada; su único paquete, `tool-cordis`, es el conjunto de herramientas de automodificación en runtime.

La estrella polar de la reagrupación: **los paquetes estrechamente agrupados comparten grupo.** Un cluster se mide — aristas de dependencia peer y co-cambio —, no es temático. Una familia de seams aislada puede estar sola como grupo pequeño; el modo de fallo a evitar es el cajón de sastre cuyo nombre no describe ningún rol único.

## Decisión

Cinco decisiones de reagrupación siguen vigentes; todos los demás grupos conservan su frontera y contenidos anteriores (el análisis de dependencias confirmó que las familias de capacidades — `shell/`, `terminal/`, `code-runtime/`, `sandbox/`, `subprocess/`, `fs/`, `lsp/`, `web/`, `skill/` y el resto — ya estaban trazadas correctamente). La sexta decisión original reunía el inicializador de proyectos del SDK, las herramientas de launcher y los paquetes JSON-RPC de runtime bajo `scaffold/`; [la eliminación de ese toolchain no publicado](../simplification/2026-08-11-remove-sdk-project-toolchain.es.md) borró las herramientas de proyecto y movió el trío de runtime superviviente a `sdk/`. El [contrato de nombres de repositorio](2026-08-11-repository-naming-contract-and-rename-ledger.es.md) posterior es dueño de los nombres de grupo `shell/`, `terminal/` y `extensions/` y de los dos nombres de paquete que esta decisión aplazó.

| Grupo | Miembros (nombres de carpeta) | Desde |
|---|---|---|
| `session/` | session-persistence, session-persistence-jsonl, session-persistence-sqlite, session-checkpoint-policy, session-projection, session-projection-cache, session-title, session-title-llm, session-title-first-prompt-llm, session-title-all-prompts-llm, session-telemetry, session-telemetry-otel | `session-persistence/` + `session-projection/` + `session-title/` + `telemetry/` |
| `interaction/` | user-questions, user-approval, permission-presets, tool-ask-user, commands, tui | `ui/` |
| `boot/` | app-boot | `ui/` |
| `guard/` | repeat-tool-reminder, timeout-policy | `guard/` + `timeout/` |
| `extensions/` | tool-cordis | `cordis/` |

- **`session/`** es el plano de datos de sesión duraderos: el seam de persistencia con sus backends y su política de checkpoints, el fold de proyección que sirve valores completos desde ese log, los títulos respaldados por el log y el informe OTel. El fold de títulos es en sí mismo portante para el lado de lectura (`session-query` depende por peer de `dsh-session-title`), así que los títulos pertenecen al plano de datos, no a un anexo de servicios derivados. El nombre llano es deliberado (prefiere nombres que un humano diría); el paquete vecino `core/session` sigue siendo el servicio vivo en memoria, mientras que este grupo es la familia duradera a su alrededor. `session-query/` sigue siendo un grupo independiente — la superficie de lectura/herramienta tiene sus propias herramientas de modelo y su backend SQLite FTS y se consume con independencia de los internals de persistencia.
- **`interaction/`** es el plano de colaboración humana más el canal de terminal que le responde: los seams de pregunta/aprobación, el preset de permisos, la herramienta `ask_user_question` orientada al modelo, el registro de comandos humanos (`plan-mode` y `command-goal` ya consumen `commands` junto con los seams de interacción), y `tui` — el canal interactivo es el provider y Consumer más ricos del plano (aristas peer a `commands` y `user-questions`), y un grupo `tui/` de un paquete gastaría un nombre de nivel superior en un solo plugin.
- **`boot/`** es un grupo de un solo paquete completo en su rol: el pegamento de boot de bin compartido que no pertenece a ningún canal ni a ningún ensamblaje (consumido por `apps/cli` y los bins de demo de `examples/`).
- **`guard/`** conserva su rol documentado, las guardas de higiene de loop, y gana el ejecutor de timeout de llamadas de herramienta, disolviendo el grupo `timeout/` de un paquete cuyo nombre chocaba con `util/timeout`.
- **`extensions/`** nombra el rol que `cordis/` oscurecía: el conjunto de herramientas con el que el agent inspecciona y monta plugins en su propio runtime vivo, y la zona de aterrizaje para futuros paquetes de automodificación.

42 grupos se convirtieron en 39; la victoria es la corrección del agrupamiento y los nombres veraces, no el recuento.

## Decisiones de nombres posteriores

El [contrato de nombres de repositorio](2026-08-11-repository-naming-contract-and-rename-ledger.es.md) resuelve los dos nombres que este movimiento aplazó deliberadamente. `@deepseek-ai/dsh-sdk-jsonrpc-server` nombra la mitad de servidor JSON-RPC del protocolo SDK de runtime. `@deepseek-ai/dsh-tool-call-timeout-policy` nombra la operación exacta limitada por la política conservando su hogar `guard/timeout-policy/`. Sus marcadores `FIXME` que bloqueaban el lanzamiento se eliminan con esos renombrados.

## Qué tocó el movimiento

Los movimientos aterrizaron como movimientos `git mv` puros, así que la detección de renombrados transporta la historia. Un movimiento de grupo tocó: las `references` relativas del `tsconfig.json` del paquete movido y la entrada de cada dependiente (incluidas las referencias de proyecto de `apps/cli`), el tsconfig agregado y los mapas de rutas, los READMEs de grupo, la tabla de jerarquía de [packages/README.md](../../../../packages/README.es.md), el mapa de layout del `AGENTS.md` raíz, los artefactos regenerados (`docs/module-graph.md`, catálogos con rutas incrustadas y las claves de importer del lockfile), y las citas `packages/...` relativas a la raíz en prosa y en scripts de puertas. Los referentes restantes de rutas de grupo (configs de workspace, globs de test, claves de lint) se encontraron mecánicamente porque las puertas de aceptación fallaron alto — la propia regla de configuración errónea del repositorio.

Un movimiento de grupo no tocó: nombres npm, imports, configs `cordis.yml`, fixtures de instantáneas, los globs de `pnpm-workspace.yaml`/`tsdown` (ambos `packages/*/*`) ni el manifest de runtime de Python — todos referencian paquetes por nombre npm.

`client/` y `host/` quedaron fuera de alcance y no cambiaron.

## Alternativas consideradas

**Cubos de dominio gruesos** (`exec/` = subprocess+sandbox+bash+pty+code-runtime, `workspace/` = fs+lsp+workspace, `orchestration/` = subagent+workflow+tasks, `knowledge/` = web+skill, `collab/` = plan+todo+goal; ~16 grupos). Rechazada: el grafo medido contradice las fusiones. `sandbox` y `subprocess` son infraestructura compartida consumida entre familias (aristas de bash ×5, fs ×5, pty, lsp, mcp y subagent), `web` ↔ `skill` tienen cero aristas, y un cubo grande reproduce el cajón de sastre de `ui/` a mayor escala.

**Nombres de capa abstractos** (`capability/`, `policy/`, `extension/`, `provider/`). Rechazada: describen igual de mal a todo plugin, y un cubo `capability/` albergaría ~50 paquetes.

**Una barrida completa de renombrados npm** (`dsh-<group>-<pkg>` para todo paquete). Rechazada: los nombres npm son planos, así que prefijar por grupo añade churn en imports, configs y fixtures sin ganancia de desambiguación; los renombrados selectivos con seguimiento FIXME cubren las colisiones reales.

**Realizar los renombrados aplazados dentro de la reorganización.** Rechazada: los renombrados multiplican los conflictos de PRs abiertos y destruyen la propiedad de revisión de movimiento puro. Los marcadores FIXME restantes los mantienen como bloqueadores de lanzamiento visibles que resolver en pequeños PRs de seguimiento.

**Una división de sesión en dos** (`session-core/` + `session-utils/`). Rechazada: query no pertenece limpiamente a ninguno de los dos lados, y `session-core` invita a la confusión con `core/session` (`dsh-session`, el servicio vivo en memoria, que permanece en `core/`).

**Una división de sesión en tres** (`session-store/` + `session-query/` + `session-utils/`). Rechazada: `session-utils/` era un anexo definido por negación («derivado, nada portante depende de él») — la forma de cajón de sastre que la estrella polar prohíbe, y además factualmente errónea (`session-query` depende por peer de `dsh-session-title`). Los nombres compuestos inventados también parecen generados por máquina; un único grupo `session/` llano dice lo que diría un humano. Query sigue siendo independiente en cualquier caso: es una superficie de lectura consumida por separado con su propio paquete de herramienta y su propio backend.

**Recomponer `ui/` como un único grupo `channels/`** (tui + jsonrpc + acp + seams de interacción + boot). Rechazada: el mismo cajón de sastre con un nombre nuevo — esos paquetes sirven a cuatro planos, el cluster medido de `jsonrpc` es la pila de cableado del SDK, y `acp/` es un transporte de automatización, no un canal humano.

**Un grupo `tui/` independiente de un solo paquete.** Rechazada: `tui` es el provider/Consumer principal del plano de interacción (aristas peer a `commands`, `user-questions`), y un nombre de nivel superior gastado en un plugin añade un grupo sin añadir información; se pliega en `interaction/`.

**Mover `app-boot` a `apps/`.** Rechazada: `apps/` es el nivel de ensamblaje sobre el nivel de paquetes, y `dsh-app-boot` es una biblioteca de nivel de paquete — colocarla en `apps/` invertiría los niveles y pondría una biblioteca de workspace fuera de los globs de build `packages/*/*`. Sigue siendo un paquete; `boot/` es su hogar completo en su rol.

**Mover `tool-cordis` a `core/`.** Rechazada: la automodificación es su propio seam de producto, con expectativas de crecer; la columna vertebral sigue siendo mínima. El grupo se llamó primero `self-evolve/`; el nombre se asentó en `extensions/` como el término más llano.

**Renombrar `context/` a `request-context/`.** Rechazada: dentro de este árbol el grupo es inequívoco in situ; el churn no está justificado.

## Consecuencias

- Las cinco familias reagrupadas aún vigentes contienen los miembros listados; los grupos `ui/`, `telemetry/`, `timeout/`, `cordis/`, `session-persistence/`, `session-projection/` y `session-title/` ya no existen. La reagrupación en sí no cambió ningún nombre npm. La eliminación posterior del toolchain del SDK cambió intencionadamente el conjunto de paquetes y restauró `sdk/` como el hogar preciso del trío de runtime del SDK. Dos marcadores FIXME fijan los renombrados aplazados restantes; un FIXME que luego resulte erróneo debe eliminarse explícitamente con su razón, nunca descartarse en silencio.
- Lo que fija el resultado: `pnpm run typecheck`, las suites unitarias de cada grupo movido, `verify-package-paths`, `verify-md-links` y el emparejamiento de traducción de todo el corpus pasan en el árbol movido; los globs de test con ámbito de grupo en `vitest.snapshot.config.ts` se reescribieron con los movimientos para que las suites recojan los mismos archivos de test que antes (un glob que falla abierto descartaría cobertura en silencio).
- Todo PR abierto que toque un archivo movido se re-basifica una vez a través del movimiento; la detección de renombrados resuelve la mayoría de los hunks mecánicamente.
- Los grupos de un solo paquete permanecen (`boot/`, `extensions/` y los existentes como `acp/`). Aceptados deliberadamente: cada uno está completo en su rol en lugar de ser un fragmento de familia, y un grupo pequeño veraz gana a una fusión nominal.
- Las carpetas de rol de `sdk/` se mapean explícitamente a sus nombres npm en `tsconfig.base.json`; el mapeo `server/` resuelve a `@deepseek-ai/dsh-sdk-jsonrpc-server`.
- Lo que esto sacrificó: nada funcional — el cambio es de navegación. La memoria muscular y los enlaces externos a rutas antiguas de GitHub se rompen, lo que es aceptable en pre-lanzamiento sin Consumers externos.
