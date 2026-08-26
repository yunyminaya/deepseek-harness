# client/ — mitad de navegador de la GUI web

[English](README.md) | Español

El lado de navegador de la GUI web de dsh: arranque del shell, comunicación navegador-anfitrión, servicios de UI compartidos y plugins de funcionalidad. Las reglas de redacción viven en [AGENTS.md](AGENTS.es.md); la mitad anfitriona es [`host/`](../host/README.es.md). Todos excepto `test-runtime` son paquetes de **producto** llamados `@deepseek-ai/dsh-client-<name>`.

| Paquete | Propósito |
|---|---|
| [`web/`](web/README.es.md) | Arranca el shell del navegador desde el grafo de entradas del cliente. |
| [`ui-renderer/`](ui-renderer/README.es.md) | Enlaza los datos de slot con React y monta la aplicación ensamblada cuando el arranque del cliente se asienta. |
| [`modules/`](modules/README.es.md) | Carga los módulos de cliente del lado del navegador. |
| [`connection/`](connection/README.es.md) | Mantiene la comunicación RPC navegador-anfitrión y la entrega de eventos. |
| [`runtime/`](runtime/README.es.md) | Proporciona servicios de cliente compartidos para sesiones, espacios de trabajo y composición de UI. |
| [`hmr/`](hmr/README.es.md) | Refresca los plugins de cliente durante el desarrollo. |
| [`locale/`](locale/README.es.md) | Proporciona preferencias de localización y diccionarios de mensajes. |
| [`test-runtime/`](../test-support/client-runtime/README.es.md) | Proporciona soporte de pruebas de repositorio compartido para los paquetes de funcionalidad de cliente. |
| [`ui-slots/`](ui-slots/README.es.md) | Define cómo las funcionalidades de UI registran y componen slots de extensión. |
| [`ui-theme/`](ui-theme/README.es.md) | Aplica el tema de color seleccionado. |
| [`ui-primitives/`](ui-primitives/README.es.md) | Proporciona controles React, iconos y renderizadores de contenido compartidos. |
| [`ui-attachment/`](ui-attachment/README.es.md) | Registra la presentación de adjuntos del compositor y de imágenes de mensaje. |
| [`ui-layout/`](ui-layout/README.es.md) | Ordena las regiones principales de la aplicación. |
| [`ui-sidebar/`](ui-sidebar/README.es.md) | Presenta la navegación de espacios de trabajo y sesiones. |
| [`ui-brand-official/`](ui-brand-official/README.es.md) | Rellena los slots genéricos de marca del navegador con el nombre oficial y las marcas. |
| [`ui-workspace/`](ui-workspace/README.es.md) | Proporciona superficies de selección y creación de espacios de trabajo. |
| [`ui-conversation/`](ui-conversation/README.es.md) | Presenta la conversación activa y su superficie de entrada. |
| [`ui-tool/`](ui-tool/README.es.md) | Compone los árboles de llamadas de Tool y las vistas claveadas por Tool. |
| [`ui-workflow-run/`](ui-workflow-run/README.es.md) | Reproduce las ejecuciones de flujo de trabajo duraderas como revelados de Chat anidados con navegación de hijos solo en vivo. |
| [`ui-goal/`](ui-goal/README.es.md) | Presenta y gestiona el objetivo actual. |
| [`ui-trajectory/`](ui-trajectory/README.es.md) | Presenta vistas alternativas de la actividad del agent. |
| [`ui-commands/`](ui-commands/README.es.md) | Proporciona descubrimiento y despacho de comandos conscientes de la sesión. |
| [`ui-input-trigger/`](ui-input-trigger/README.es.md) | Coordina las sugerencias en línea de comandos y referencias. |
| [`ui-skill/`](ui-skill/README.es.md) | Añade referencias de skill a las sugerencias en línea. |
| [`ui-reference/`](ui-reference/README.es.md) | Fuente unificada de referencias Web `@file` / `@session`. |
| [`ui-subagent/`](ui-subagent/README.es.md) | Proporciona navegación de subagentes, estados de transcript hijo y referencias en línea. |
| [`ui-jobs/`](ui-jobs/README.es.md) | Lista los trabajos en segundo plano de esta sesión en la cabecera de la conversación. |
| [`ui-model-selection/`](ui-model-selection/README.es.md) | Proporciona selección de modelo en las superficies de conversación. |
| [`ui-permission/`](ui-permission-presets/README.es.md) | Configura los permisos predeterminados y conmuta el acceso de la sesión actual. |
| [`ui-plan/`](ui-plan/README.es.md) | Presenta el estado activo del modo plan y su control de salida. |
| [`ui-settings-plugins/`](ui-settings-plugins/README.es.md) | Es dueño de la sección de ajustes de Plugins, su punto de extensión de pestañas y las tarjetas configurables de plugins del plano anfitrión. |
| [`ui-user-questions/`](ui-user-questions/README.es.md) | Presenta las preguntas interactivas solicitadas por el agent. |
| [`ui-agent-preset/`](ui-agent-preset/README.es.md) | Selecciona el preset de agent de una sesión y redacta composiciones de preset. |
| [`ui-settings/`](ui-settings/README.es.md) | Aloja la interfaz de ajustes y sus áreas de extensión. |
| [`ui-settings-general/`](ui-settings-general/README.es.md) | Proporciona la sección de ajustes generales. |
| [`ui-settings-models/`](ui-settings-models/README.es.md) | Proporciona la configuración de providers de modelo y la incorporación de DeepSeek. |
| [`ui-settings-plugin-inventory/`](ui-settings-plugin-inventory/README.es.md) | Contribuye la pestaña de inventario del Loader de Host de solo lectura a los ajustes de Plugins. |

Cada referencia hija es dueña de su contrato y su comportamiento detallado. El [estándar del sistema de slots](../../.agents/notes/implemented/architecture/2026-07-22-slot-type-chain-implementation.es.md) y la [nota de arquitectura del cliente web](../../.agents/notes/implemented/architecture/2026-07-19-gui-web-client-architecture.es.md) son dueños de la composición entre paquetes y de las decisiones de carga.

La referencia de subsistema es [client-modules.md](../../docs/subsystems/client-modules.es.md); el [estándar del sistema de slots](../../.agents/notes/implemented/architecture/2026-07-22-slot-type-chain-implementation.es.md) es el modelo de slots definitivo, y la [nota de arquitectura del cliente web](../../.agents/notes/implemented/architecture/2026-07-19-gui-web-client-architecture.es.md) es dueña de la cadena de carga y de la capa de objetos.
