# @deepseek-ai/dsh-client-ui-workflow-run

[English](README.md) | Español

El plugin de navegador que reconstruye las ejecuciones durables de flujos de trabajo de nivel superior como nodos de Chat independientes. Consume los cuatro eventos de Session `tool-workflow/*` propiedad de [`dsh-tool-workflow`](../../workflow/tool-workflow/README.es.md), registra una `ConversationNodeDefinition` y renderiza a través del slot con clave `conversation.chat.node` sin cambiar la tarjeta de herramienta de flujo de trabajo existente.

## Estado durable y repetición

`tool-workflow/run-start` crea un Context con clave por `runId`; los inicios de miembros, los finales de miembros y el final de la ejecución actualizan ese Context en orden de registro. Una cola de historial que contiene solo actualizaciones permanece pendiente hasta que una página anterior aporta el inicio único, tras lo cual el prepend, la repetición completa y el append en vivo producen el mismo estado. Un Turn o Step cerrado con eventos terminales ausentes presenta la ejecución o los miembros afectados como interrumpidos sin cambiar el resultado de la herramienta.

Los grupos de fase provienen solo de los miembros que realmente iniciaron. Las cadenas de fase exactas comparten grupo, una fase omitida es distinta de la cadena vacía, y la resolución cambia el estado sin eliminar ni reordenar miembros.

## Presentación y navegación

La ejecución y cada fase son disclosure controls en todos los estados. Un montaje abre los niveles en ejecución, fallidos, cancelados e interrumpidos y cierra los niveles completados del todo; el usuario puede entonces alternar cada nivel con la fila completa, Enter o Space. Las actualizaciones ordinarias de ejecución conservan la elección actual, el primer borde anómalo abre una vez, la finalización normal cierra una vez, y una fase completada más la ejecución exterior se abren de nuevo cuando un nuevo miembro en ejecución inicia bajo la misma clave de fase. Si un ciclo nuevo y limpio completo llega en un solo render mientras la ejecución sigue activa, la fase termina plegada pero la ejecución exterior abre una vez para exponer su resumen actualizado. La finalización actualiza el estado visible de inmediato pero retrasa su cierre automático mientras el foco permanezca dentro del contenido. `WorkflowRunPanel` es dueño de las elecciones de fase, así que cerrar y reabrir la ejecución exterior no las reinicia; un remount del renderizador reconstruye cada elección inicial a partir de hechos durables. La ejecución usa una fila de 32 píxeles `--dsw-alias-bg-module-platform` con chevrons persistentes a la derecha/abajo y un punto de estado en línea más el texto de estado, sin badge. Las fases usan filas de disclosure de 32 píxeles con título y recuento de miembros en el área principal flexible y una cola fija de estado agregado preciso, sin otro punto. Los miembros usan un slot de punto de 16 píxeles, un área de nombre que trunca y una columna de estado fija de 64 píxeles.

Un miembro abre una Session hija solo mientras todos los hechos actuales coinciden: el miembro está en ejecución, el id hijo está en la lista ordinaria de Sessions, la fila tiene `origin: 'subagent'`, su `parentId` es la Session actual y la fila de la lista sigue en ejecución. El texto de miembro subrayado es la única affordance de navegación visible; el foco del teclado dibuja un anillo de dos píxeles en el color principal de negocio alrededor del área del nombre, mientras la copia de estado sigue siendo `Running`. El componente llama solo a la acción ordinaria inyectada `sessions.open(id)`; las filas remotas, solo direccionadas, de padre equivocado o terminales permanecen no interactivas.

## Composición

El paquete registra su Definition, su diccionario de locale y el renderizador `workflow-run` como efectos de Cordis. Eliminar la entrada de cliente retira las tres contribuciones. El bundle Web distribuido incluye el plugin después de `ui-conversation` y `ui-tool`.

## Experiencia del modelo

Ninguna, ya que este paquete renderiza hechos durables de Session para humanos y no añade prompt, schema de herramienta, contenido de solicitud ni resultado visible por el modelo.

#### Efecto en la caché KV

Ninguno.

## Limitaciones conocidas y trabajo diferido

- Solo las llamadas de nivel superior a través de `dsh-tool-workflow` producen estos registros; las llamadas anidadas de Code Mode y los consumidores directos de `WorkflowEngine` no.
- La navegación es intencionadamente solo en vivo. Los miembros terminales permanecen visibles para revisión pero nunca exponen un abridor de sesión fría desde este nodo.
- El nodo muestra solo identidad de ejecución, fase y miembro, y estado; los scripts, salidas, errores, logs, uso, topología estática y controles quedan fuera de esta superficie.
