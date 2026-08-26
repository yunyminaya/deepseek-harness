# Subsistemas

[English](README.md) | Español

Una página por subsistema de DeepSeek Harness: qué es, las estructuras de datos que mueve y — donde un servicio `ctx` o un ámbito de eventos lo respalda — una sección generada de **Cordis API** con su referencia de servicios y eventos. La carpeta complementa a [architecture.md](../architecture.es.md), que describe el *comportamiento* entre subsistemas (el mapa de servicios, el ciclo de vida de sesión/turno/paso, la taxonomía de eventos); cada página aquí es la referencia del vocabulario y el cableado de un subsistema.

| Página | Es dueña de |
|---|---|
| [core.md](core.es.md) | cómo `packages/core` controla el agent loop (bucle del agente): la descripción del loop paquete por paquete, la creación y propiedad de agents (`AgentHandle`), los contratos de entrega/cancelación/interceptación del handle `Agent`, y los patrones de tipo de todo el repositorio (`…Map → derived-union`, ids con marca) |
| [llm-streaming.md](llm-streaming.es.md) | los tipos de conversación de `packages/llm` — `Message`/`ContentBlock`, la solicitud de modelo ensamblada, el protocolo de red `StreamChunk` y el contrato del adaptador, `BlockAssembler`, y el contrato de provider `LlmAdapter` |
| [token-meter.md](token-meter.es.md) | mediciones de reproducción escalares y posicionales inmutables con revisiones de log consumido |
| [scope.md](scope.es.md) | identidad de registro con ámbito, carriers de dispatch y el contexto `Scope` propiedad del registro |
| [typert.md](typert.es.md) | descriptores de invocación remota, declaraciones de lookup/Context, registros de Typert y las fronteras de Host Gateway/API de Client |
| [goal.md](goal.es.md) | identidad de goal persistida, instantáneas de ciclo de vida, activación, registros de cambio y atribución de ronda |
| [schedule.md](schedule.es.md) | registros de recordatorios locales a la sesión, transiciones duraderas, vistas activas y entrega en conversación ordinaria |
| [commands.md](commands.es.md) | el servicio de registro de comandos humanos: definiciones, descubrimiento de adaptadores, invocación directa, resultados y vistas de análisis |
| [session.md](session.es.md) | el catálogo completo de variantes de `SessionEventMap`, `TurnTrigger`/`TurnEndReason`, `deriveMessages()`, el recinto de ejecución y los eventos independientes |
| [persistence.md](persistence.es.md) | el seam de durabilidad: `SessionPersistence`, backends JSONL + SQLite, `session/flush`, recuperación de cuelgues, `SessionHeader` |
| [settings.md](settings.es.md) | el seam de ajustes de usuario: registro de `SettingsNamespace`, resolución en capas (defaults → `base` de composición → documento del usuario), scopes de propietario, commits en caliente |
| [credentials.md](credentials.es.md) | el seam de credenciales: referencias `CredentialRef` (nunca valores) en la configuración, resolución por operación, `CredentialInfo` seguro para UI, capas de fuente del provider |
| [session-query.md](session-query.es.md) | registros lógicos, lecturas acotadas de eventos exactos, trazas de relaciones, filtros/documentos semánticos y páginas de resultados de texto completo |
| [feedback.md](feedback.es.md) | registros de feedback por mensaje ligados al ciclo de vida, versiones optimistas, persistencia sidecar y el contrato de Host Remote |
| [session-title.md](session-title.es.md) | instantáneas de título duraderas, seqs citados de mensajes fuente y el contrato asíncrono del provider |
| [session-reference.md](session-reference.es.md) | referencias estructuradas entre sesiones: `SessionReferenceInput`/`Candidate`, contextos de mensaje preparados, la taxonomía estable de errores |
| [system-prompt.md](system-prompt.es.md) | contexto por ensamblaje, resultados de tool-provider, secciones de prompt y ensamblaje cooperativo |
| [tools.md](tools.es.md) | todos los campos de `ToolDefinition`, el DSL de schema, `ToolExecution`/`ToolResult`, los tipos de UI de presentación de herramientas y el pipeline de ejecución protegido |
| [user-questions.md](user-questions.es.md) | el seam de preguntas/respuestas humanas respaldado por UI: `AskUserQuestionRequest`, vocabulario de respuestas/opciones, API del provider, taxonomía de errores |
| [approval.md](approval.es.md) | el seam de aprobación de usuario de un solo uso: `ApprovalRequest`, `ApprovalOutcome`, política por sesión, eventos de auditoría y contratos de quien responde |
| [attachment.md](attachment.es.md) | identidad y metadatos de imagen duraderos, entradas de validación, lecturas verificadas y el seam `AttachmentStore` |
| [shell.md](shell.es.md) | el seam del ejecutor de bash: `ShellExecRequest`/`Spec`, `ShellRunResult`, handles de `ShellProcess` en segundo plano |
| [subprocess.md](subprocess.es.md) | el seam de subprocess: `SubprocessSpawnSpec` totalmente explícito, lectores de salida por offset, `SubprocessOutcome` sin clasificar y el vocabulario gestionado de entorno `DSH_*` |
| [terminal.md](terminal.es.md) | ids de terminal persistentes, contratos de backend/sesión, disposición de envío, lecturas acotadas e instantáneas visibles para el propietario |
| [sandbox.md](sandbox.es.md) | resolución de política por sesión y el seam de confinamiento de procesos: modos de efecto de archivo, políticas de ejecución/provider, `ConfinedArgv`, errores de aplicación y fail-closed |
| [code-runtime.md](code-runtime.es.md) | el seam de ejecución de código: `CodeRunRequest`/`Result`, namespaces de binding, logs capturados, la taxonomía `CodeRunFailure` |
| [extensions.md](extensions.es.md) | Cordis Plugins y Packages dinámicos con versiones, activación de Host/Client, aprobación, inspección de runtime y teardown de ciclo de vida |
| [filesystem.md](filesystem.es.md) | el seam de sistema de archivos: `FsTarget`, resultados de lectura/escritura/edición, estado de archivo observado, `FsErrorCode` |
| [lsp.md](lsp.es.md) | el seam de navegación LSP: `LspQueryRequest`/`Result`, `LspProvider`/`Service`, cuatro operaciones, `LspError` |
| [skills.md](skills.es.md) | el servicio de skills: prioridad de descubrimiento, `SkillSummary`/`SkillDefinition`, catálogo de prefijo de sesión, carga de `skill` orientada al modelo |
| [compaction.md](compaction.es.md) | el seam de compactación: los eventos de sesión `compaction/*`, `CompactionResult`, la interfaz `CompactionEngine` |
| [subagent.md](subagent.es.md) | el seam de subagentes: el registro de providers con nombre, `SubagentStartRequest`/`Result`/`Run`, la división de capacidades entre tiempo de arranque y runtime |
| [agent-team.md](agent-team.es.md) | Agent Teams: identidad de Lead implícita, teammates continuables con nombre, mailbox durable entre pares y DAG de tareas compartido |
| [web.md](web.es.md) | el seam de acceso web: `WebSearchRequest`/`Result`, `WebFetchRequest`/`Result`, `WebFetchBody`, disponibilidad del provider, `WebError` |
| [spill.md](spill.es.md) | el seam de almacenamiento spill: `SaveTextSpill`, `SpillOwner`/`SpillSource`, `SpillRef`, el `SpillLocator` con marca |
| [workflow.md](workflow.es.md) | el seam de flujos de trabajo: `WorkflowStartRequest`, `WorkflowMeta`, `WorkflowRun`/`Result`, los payloads de eventos `workflow/*`, la fatalidad de `WorkflowError` |
| [jobs.md](jobs.es.md) | el runtime de jobs en segundo plano: `JobId`s con marca, el contrato del productor, vistas del consumer y el comportamiento del servicio `ctx.jobs` |
| [permission-presets.md](permission-presets.es.md) | la capa de preajustes de permiso: `PresetSpec`/`PresetOption`, el estado `custom` derivado, el evento solo-log `permission/preset` |
| [plan.md](plan.es.md) | el modo plan: el estado solo-log `plan/mode`, el flush de selección pendiente, `PlanModeConfig`, el arco de revisión `exit_plan_mode` |
| [invariants.md](invariants.es.md) | el registro de invariantes de runtime: `Config` de selección, `InvariantInstaller`/`InvariantFailure`, el contrato de companion vacío |
| [web-server.md](web-server.es.md) | el carrier HTTP: `WebRouteKind`/`WebRoute`, orden de coincidencia, el asiento fallback reclamable, taps de índice |
| [storage.md](storage.es.md) | el subsistema de almacenamiento: el contrato del backend (`StorageBackend`), `StorageForms`, `DomainSpec`/`Domain`, `domain/changed` |
| [workspace.md](workspace.es.md) | el registro de workspaces: `Workspace`/`WorkspaceId`, registro y resolución, la relación con el `cwd` de la sesión |
| [client-modules.md](client-modules.es.md) | la tabla de plugins web: declaraciones `dsh.client`, composición de red `WebBootGraph`, la ruta del bundle y el tap de índice |
| [session-projection.md](session-projection.es.md) | el seam de proyección: `SessionProjectionMap`, la unidad pura `ProjectionDefinition`, el corte consistente de `ProjectionSnapshot`, el feed de cambios |
| [session-telemetry.md](session-telemetry.es.md) | el seam de capacidad de reporte de sesión saliente: `SessionTelemetryRecord`/`SessionTelemetrySeverity`, el contrato `SessionTelemetrySink` y el waterfall de redacción `session-telemetry/record` |

> Las declaraciones de tipos y su JSDoc en estas páginas son equivalentes a la fuente y se verifican contra desviaciones con `pnpm run verify-type-equiv` (consulta [development.md](../development.es.md#documenting-types-verbatim-ts-type-equiv)). Los bloques ordinarios preservan declaraciones completas; los bloques `public-api` preservan declaraciones de clases públicas sin el cuerpo. Los servicios y eventos de Cordis usan la sección generada de **Cordis API** de cada página.
