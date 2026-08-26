# Paquetes

[English](README.md) | Español

Ámbito npm: `@deepseek-ai/dsh-*`; las subclases de `Service` de Cordis y los plugins de función contribuyen a través de `ctx.effect()`, `ctx.on()` o `ctx.waterfall()`. Reglas: [paquete](AGENTS.md), [raíz](../AGENTS.md#conventions).

## Jerarquía

Los grupos contienen `packages/<group>/<pkg>/`; los nombres siguen siendo `@deepseek-ai/dsh-<pkg>`. **Los READMEs de grupo son dueños de los mapas de paquete/clave de ctx.**

| Grupo | Rol | Expectativa de publicación |
|---|---|---|
| [`core/`](core/README.es.md) | Eje de la API de producto: sesiones, prompts, herramientas, servicios de agent y el loop concreto | Producto — API estable |
| [`api/`](api/README.es.md) | Ensamblaje del BFF remoto y gateway RPC de Typert | Producto — API estable |
| [`typert/`](typert/README.es.md) | Generación del grafo de tipos, carga de artefactos y registro de runtime | Producto — API estable |
| [`goal/`](goal/README.es.md) | Persistencia y ciclo de vida de los objetivos de la misma sesión | Producto — API estable |
| [`schedule/`](schedule/README.es.md) | Seguimientos programados locales a la sesión | Producto — API estable |
| [`feedback/`](feedback/README.es.md) | Comentarios humanos | Producto — API estable |
| [`identity/`](identity/README.es.md) | Identidad anónima compartida | Producto — API estable |
| [`llm/`](llm/README.es.md) | Familia de capacidad LLM: el servicio abstracto + adaptadores de provider | Producto — API estable |
| [`e2b/`](e2b/README.es.md) | Providers E2B | POC |
| [`subprocess/`](subprocess/README.es.md) | Familia de capacidad de subprocesos: Service Definition + provider local de árbol de procesos | Producto — API estable |
| [`shell/`](shell/README.es.md) | Familia de capacidad de Bash: seam de executor, implementación local, herramienta visible para el modelo | Producto — API estable |
| [`terminal/`](terminal/README.es.md) | Familia de capacidad PTY persistente: sesiones con ámbito de dueño, implementación local y herramientas visibles para el modelo | Producto — API estable |
| [`code-runtime/`](code-runtime/README.es.md) | Familia de capacidad de ejecución de código: Service Definition + provider de hilo de trabajo + Consumer de Code Mode | Producto — API estable |
| [`sandbox/`](sandbox/README.es.md) | Seam de confinamiento de procesos; backends bwrap/Landlock/Seatbelt | Producto — API estable |
| [`fs/`](fs/README.es.md) | Familia de capacidad de sistema de archivos: seam, implementación local, herramientas de archivos visibles para el modelo, herramientas de descubrimiento basadas en bash | Producto — API estable |
| [`lsp/`](lsp/README.es.md) | Familia de capacidad LSP: seam, provider genérico de stdio y la herramienta `lsp` | Producto — API estable |
| [`skill/`](skill/README.es.md) | Familia de capacidad de skill: el registro de providers, el provider local y el catálogo/loader visible para el modelo | Producto — API estable |
| [`compaction/`](compaction/README.es.md) | Familia de capacidad de compactación: Service Definition + provider básico + Consumer de comandos | Producto — API estable |
| [`context/`](context/README.es.md) | Contexto de solicitud visible para el modelo, incluidas las instrucciones del espacio de trabajo y el contexto de tiempo | Producto — API estable |
| [`subagent/`](subagent/README.es.md) | Familia de capacidad de subagente: el contrato de registro de providers y la herramienta de delegación visible para el modelo | Producto — API estable |
| [`jobs/`](jobs/README.es.md) | Runtime genérico de trabajos en segundo plano y herramientas de control `job_*` visibles para el modelo | Producto — API estable |
| [`experimental/`](experimental/README.es.md) | Prototipos privados y plugins solo internos | Sin publicar |
| [`workflow/`](workflow/README.es.md) | Seam de flujo de trabajo, motor de hilo de trabajo y herramientas `workflow`/`ralph` visibles para el modelo | Producto — API estable |
| [`web/`](web/README.es.md) | Familia de capacidad web: seam, implementaciones de providers de búsqueda/obtención y herramientas web visibles para el modelo | Producto — API estable |
| [`attachment/`](attachment/README.es.md) | Identidad duradera de adjuntos, validación, almacenamiento local direccionado por contenido | Producto — API estable |
| [`spill/`](spill/README.es.md) | Familia de capacidad de spill: seam de almacenamiento, implementación local, política de spill de resultados de herramientas | Producto — API estable |
| [`todo/`](todo/README.es.md) | La herramienta `todo_write` visible para el modelo | Producto — API estable |
| [`plan/`](plan/README.es.md) | Estado de colaboración de planes con un comando de entrada directo y una salida revisada | Producto — API estable |
| [`preset/`](preset/README.es.md) | Composición de agent por sesión a partir de archivos `cordis.yml` de preset | Producto — API estable |
| [`guard/`](guard/README.es.md) | Guardas de higiene del loop: recordatorios consultivos de llamadas repetidas + el aplicador de plazos `tools/execute` | Producto — API estable |
| [`bundle/`](bundle/README.es.md) | Capas de parche instalables `dsh --profile` | Producto — API estable |
| [`extensions/`](extensions/README.es.md) | Automodificación del runtime del agent: inspección en vivo de plugins/servicios y montaje/desmontaje de plugins escritos por el modelo ([diseño](../.agents/notes/implemented/feature/2026-07-08-self-referential-cordis-toolset.es.md)) | Producto — API estable |
| [`hooks/`](hooks/README.es.md) | Puentes de hook + la biblioteca compartida de protocolo de cableado Claude Code / Codex | Producto — API estable |
| [`session/`](session/README.es.md) | Plano de datos duradero de sesión: seam de persistencia + backends JSONL/SQLite, seam de proyección, títulos respaldados por logs, informes de sesión | Producto — API estable |
| [`session-query/`](session-query/README.es.md) | Familia de recuperación de sesiones: corpus lógico, lecturas acotadas, linaje, relaciones de eventos, filtrado semántico y búsqueda de texto completo en SQLite | Producto — API estable |
| [`settings/`](settings/README.es.md) | Seam de ajustes de usuario + provider respaldado por archivo | Producto — API estable |
| [`credentials/`](credentials/README.es.md) | Seam de referencia/registro de credenciales + provider de env sobre `.env` + flujos de autorización | Producto — API estable |
| [`storage/`](storage/README.es.md) | Centro de almacenamiento no de sesión + backends + formulario de dominio | Producto — API estable |
| [`workspace/`](workspace/README.es.md) | Entidad de espacio de trabajo | Producto — API estable |
| [`sdk/`](sdk/README.es.md) | SDK de runtime fuera de proceso: protocolo JSON-RPC, cliente TypeScript y plugin de servidor | Producto — API estable |
| [`acp/`](acp/README.es.md) | Servidor de Agent Client Protocol solo de automatización | Producto — API estable |
| [`interaction/`](interaction/README.es.md) | Plano de colaboración humana: seams de aprobación/interacción, preset de permisos, comandos, herramienta ask-user | Producto — API estable |
| [`boot/`](boot/README.es.md) | Pegamento compartido de arranque de binarios de la app | Producto — API estable |
| [`host/`](host/README.es.md) | Mitad host de la GUI web: gateway de API + servidor de rutas HTTP | Producto — API estable |
| [`client/`](client/README.es.md) | Mitad de navegador de la GUI web: shell, cableado, servicios de objetos, slots, plugins `ui-*` | Producto — API estable |
| [`examples/`](examples/README.es.md) | Bundles de demostración (agent-spine + bins CLI/ACP/JSON-RPC) que cargan las hojas | Soporte — infraestructura de ejemplos |
| [`test-support/`](test-support/README.es.md) | Infraestructura de soporte (testkits, invariants, replay, smokes del Loader) | Soporte — expectativas de compatibilidad menores |
| [`util/`](util/README.es.md) | Utilidades de bajo nivel sin dependencias compartidas entre grupos (`Branded<B>`, helpers de home/rutas del Harness, timeout, retención) | Soporte — pequeñas, estables, sin dependencias del harness |

Los paquetes nuevos se incorporan a grupos existentes; los grupos nuevos actualizan su README y esta tabla.

## Dependencias

El grafo de dependencias se genera: [docs/module-graph.md](../docs/module-graph.es.md) (`pnpm run gen-module-graph`, con control de frescura en CI).

**Los plugins de extensión dependen de Service Definitions, nunca de providers concretos.** `dsh-agent-loop` es intercambiable; los plugins de UI, hook y herramienta usan `dsh-agent`. Los bundles de composición, incluido `dsh-agent-spine-demo`, pueden depender de plugins de spine. Las capacidades separan los roles de Service Definition / Service Provider / Consumer cuando evolucionan de forma independiente; ver [capability seams](../.agents/notes/implemented/architecture/2026-06-13-capability-seams.es.md).

Los READMEs de paquetes cubren el propósito, las APIs, los puntos de extensión y la [Model Experience](../docs/cookbook/adding-a-package.es.md#4-write-the-package-readme) salvo que estén en la [allowlist de omisión](../scripts/verify-package-readme-model-experience.ts) independiente del modelo. También llevan `## Known Limitations and Deferred Work` o usan su [allowlist](../scripts/verify-package-readme-limitations.ts).
