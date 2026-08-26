# Agent Note: Pestañas propiedad de la funcionalidad en los ajustes de Plugins

Status: implemented

[English](2026-08-11-plugin-settings-tabs.md) | [中文](2026-08-11-plugin-settings-tabs.zh.md) | Español

## Problema

La configuración de plugins y el inventario de Loader de solo lectura registraban cada uno una `settings.section` de nivel superior. Describían el mismo dominio de Plugins pero ocupaban dos filas de navegación, dividían la búsqueda y la configuración en páginas no relacionadas y no daban al shell de ajustes una forma razonada de presentarlas juntas. Combinar sus componentes directamente haría en cambio que un plugin de funcionalidad importara y fuera dueño del ciclo de vida de datos de la otra funcionalidad.

## Decisión

`@deepseek-ai/dsh-client-ui-settings-plugins` es dueño de la única contribución de `settings.section` con id `plugins`. Renderiza el título compartido y el chrome compacto de pestañas, declara el slot de lista con ámbito raíz `settings.plugins.tab` y proyecta en sus pestañas el id, el orden y la etiqueta que sigue a la locale de ese registro. El tipo canónico del slot vive en `ui-settings`, de modo que un contribuyente de pestaña depende del contrato del dominio de Ajustes y no de otro plugin de funcionalidad.

El dueño de la sección contribuye una pestaña `configurable` que declara la lista anidada existente `settings.plugin.item`. Las tarjetas de configuración conservan sin cambios sus enlaces de namespace, su estado de borrador, su validación y sus escrituras. `@deepseek-ai/dsh-client-ui-settings-plugin-inventory` contribuye una pestaña `all` a `settings.plugins.tab`; su observer del Loader del Host, su namespace Remote generado, su DTO y sus semánticas de búsqueda permanecen sin cambios. Las entradas de inventario deshabilitadas omiten de los resúmenes y detalles el estado de runtime desmontado redundante, mientras que las entradas habilitadas siguen exponiendo su fase de Cordis.

La primera pestaña ordenada se selecciona por defecto. Una pestaña se monta solo al ser seleccionada por primera vez y luego permanece montada pero oculta mientras la sección de Plugins siga montada. Esto retrasa el RPC de inventario hasta que el usuario abre **Plugin list** y preserva borradores, texto de búsqueda, estado de divulgación y la instantánea obtenida mientras se cambia de pestaña. Cerrar Ajustes desmonta la sección, de modo que al reabrir se obtiene una instantánea de inventario fresca cuando esa pestaña se selecciona de nuevo.

Ambos registros usan `ctx.slots.inject()`. Si el declarante de la sección se descarga, la declaración de pestaña y todas sus contribuciones colapsan con él; la redeclaración permite a cada funcionalidad re-registrarse sin una importación estática ni una dependencia de orden de activación.

## Alternativas consideradas

**Mantener dos filas de navegación de Ajustes y solo renombrarlas.** Rechazado porque la duplicación es estructural, no de redacción: ambas páginas siguen representando el mismo dominio de Plugins y compiten por espacio de navegación.

**Importar el componente de inventario en `ui-settings-plugins`.** Rechazado porque el plugin de configuración sería entonces dueño de la dependencia Remote y del ciclo de vida de otro plugin. También convertiría una contribución de navegador opcional en una dependencia a nivel de paquete.

**Codificar las dos etiquetas y componentes de pestaña en el dueño de la sección.** Rechazado porque una tercera funcionalidad exigiría editar al dueño, y el teardown de HMR podría dejar chrome para una contribución que ya no existe. El registro de slots ya proporciona identidad, ordenamiento, localización y semánticas de cascada.

**Mover la agregación de Plugins a `ui-settings-general`.** Rechazado porque el shell de Ajustes es dueño de la navegación genérica y del chrome modal, no del contenido de funcionalidades. Añadir pestañas específicas de Plugins allí haría de cada vista futura de Plugins un cambio de shell.

## Consecuencias

Ajustes tiene una fila de navegación de Plugins, ordenada antes de Agent Presets, con las pestañas **Plugin configuration** y **Plugin list**. Agent Presets sigue siendo una sección independiente porque edita composiciones de agent por sesión en lugar del árbol de Loader en vivo del Host.

La propiedad de la funcionalidad sigue siendo explícita: `ui-settings-plugins` es dueño de la página de Plugins y de las tarjetas editables, `ui-settings-plugin-inventory` es dueño de la vista de inventario de solo lectura, y la ruta Host/RPC no cambia. Una vista nueva de Plugins puede unirse registrando una contribución `settings.plugins.tab`.

La agregación depende de que el dueño de la sección esté compuesto: sin `ui-settings-plugins`, `ui-settings-plugin-inventory` espera una declaración de pestaña y no renderiza nada. Esa es una dependencia de composición intencional transportada por el registro de slots en lugar de una importación de paquete estática.
