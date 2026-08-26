# @deepseek-ai/dsh-client-ui-settings-plugin-inventory

[English](README.md) | Español

Pestaña **Lista de plugins** de solo lectura para los ajustes web. El plugin de navegador registra una contribución localizada `settings.plugins.tab` con el id `all`; la sección Plugins es la dueña de la entrada de navegación y del chrome de la pestaña. No realiza ninguna lectura Remote durante la activación del plugin. Seleccionar la pestaña por primera vez la monta y llama perezosamente a `ctx.remote.pluginInventory.list()` a través de [`api-remotes`](../../api/remotes/README.es.md).

La pestaña renderiza un catálogo de dos columnas con búsqueda de tarjetas desplegables compactas. Cada tarjeta colapsada usa el nombre corto del módulo como título y una etiqueta pequeña de habilitación efectiva; las entradas habilitadas también muestran un punto de estado de fibra raíz de color. Expandir una tarjeta revela el id de entrada de su árbol Loader sin una etiqueta de campo redundante, seguido de la configuración efectiva y, para las entradas habilitadas, el estado de Cordis. Las entradas deshabilitadas omiten el estado de runtime no montado redundante. El id de entrada sigue siendo la clave de React, la identidad del desplegable, el valor de detalle y un objetivo de búsqueda adicional; nunca se clasifica por su forma de cadena. Los estados de carga, vacío, sin coincidencias y de fallo genérico permanecen locales al componente montado, y una lectura fallida puede reintentarse sin exponer los detalles del transporte. El registro usa `ctx.slots.inject()`, así que sigue la declaración tardía de la pestaña, la redeclaración, los cambios de locale y el desmontaje sin importar al dueño de la sección.

## Model Experience

Ninguno: este paquete solo visualiza una instantánea de despliegue propiedad del Host en los ajustes del navegador y no registra nada orientado al modelo.

#### Efecto de KV Cache

Ninguno; este paquete ni ensambla ni envía una petición de provider.

## Limitaciones conocidas y trabajo pendiente

- **Una instantánea por montaje de Ajustes o reintento** — la pestaña no se suscribe a los cambios del Loader ni vuelve a buscar automáticamente tras reconectar; cambiar de pestaña conserva la instantánea actual, mientras que reabrir Ajustes obtiene una nueva.
- **Vista de solo lectura del Loader** — la búsqueda local no añade procedencia, diagnóstico de activación del navegador actual, agrupación por origen ni controles de mutación de plugins.
