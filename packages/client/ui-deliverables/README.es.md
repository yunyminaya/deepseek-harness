# @deepseek-ai/dsh-client-ui-deliverables

[English](README.md) | Español

Propietario de la funcionalidad de archivos producidos y referencias clicables. El lado Node registra la guía de respuesta final en el registro del system-prompt; el lado del navegador registra la fila de entregables con la que termina un turno acabado en el hueco `conversation.chat.turnTail` de la vista de chat y enlaza las referencias de código en línea coincidentes de la prosa de cierre. El parche Web distribuido es la única composición que carga este paquete. Eliminar su única entrada de cordis.yml elimina a la vez la guía, la fila y los enlaces de la prosa.

`deliverablesDefinition` pliega las llamadas de mutación correctas de cada Turn en el `DeliverablesTurnData` publicado por el motor; `producedForClosing` lee esos datos con la seq de Assistant de cierre. El vocabulario son los `locations` de seguimiento propios de las herramientas de mutación, nunca la prosa de cierre: un archivo producido aparece listado mencione o no el modelo su nombre. Una mutación se reconoce por la intención de render, no por el nombre de la herramienta —una tarjeta de diff, o una tarjeta genérica cuyo `kind` es `edit` (la forma que presenta el insert de `str_replace_editor`)—, de modo que una nueva herramienta de mutación se incorpora declarando lo que hace. Las lecturas, las eliminaciones y las llamadas fallidas no aportan nada; una ruta aparece una vez por Turn en el orden de primera aparición. El índice Conversation Location es el dueño de la pertenencia a Turn, así que un Turn que muta y luego termina sin texto de contenido no puede derramarse en la fila del siguiente Turn.

`ProducedFiles` renderiza la fila entre el cuerpo del mensaje de cierre y su pie de IconActions: una etiqueta discreta y un carril de archivo medido. Muestra el prefijo inicial más largo que cabe (hasta seis chips; texto del basename, ruta completa como `title`) mientras reserva el ancho exacto localizado de `+ N files`, de modo que el resto permanece visible sin saltos de línea ni scroll horizontal. Cada chip se abre mediante el `openFile` suministrado por el propietario —el mismo abridor del Host que usan las filas de herramienta, con la vista de chat resolviendo las rutas relativas contra el cwd de la sesión—. Cuando los archivos están ocultos, una acción de segunda línea **Mostrar en carpeta** abre el workspace de la sesión a través de esa misma ruta del propietario solo mientras la página sea loopback y el handshake actual del Host reporte `canOpenPath`; los Hosts Web remotos directos y los Hosts Linux headless o de contenedor omiten la acción por defecto. Fundamento de diseño: la [Agent Note sobre enlaces de archivos del workspace](../../../.agents/notes/implemented/feature/2026-07-31-web-workspace-file-links.es.md).

La prosa de cierre lleva el mismo vocabulario. Este plugin proporciona el servicio `chatFileMentions` que la vista de chat consulta por cada mensaje de cierre: `producedFileMentions` resuelve un token de código en línea por ruta exacta, o por ser exactamente el basename de exactamente una ruta producida —un basename que comparten dos rutas permanece inerte en lugar de adivinar, de modo que un enlace de mención nunca puede abrir el archivo equivocado ni dar un 404—. Una mención resuelta conserva su chip de código y adopta el lenguaje de enlace de la hoja markdown —azul de enlace en reposo, subrayado al pasar el ratón, exactamente como el código en línea promocionado a URL—, con la ruta completa como `title`; las menciones nunca se renderizan dentro de anclas ni de texto en streaming. Registro de decisión: la [Agent Note sobre menciones de archivos en línea](../../../.agents/notes/implemented/feature/2026-08-07-web-inline-file-mentions.es.md).

El lado Node registra la sección estática de system-prompt `ui:deliverable-file-references`. Pide al modelo que mencione los archivos principales que creó o modificó correctamente y que escriba esos y cualquier otra referencia de archivo cambiado como código en línea Markdown, usando la ruta exacta de la herramienta de archivo o un basename solo cuando sea único dentro del Turn. La guía hace explícita la sintaxis aceptada por el renderizador; no gobierna discusiones de rutas no relacionadas ni amplía el vocabulario de mutaciones correctas del renderizador.

## Experiencia del modelo

### Guía de referencias de archivo clicables

#### Lo que ve el modelo

Un párrafo fijo instruye al modelo a nombrar en su respuesta final los archivos principales de llamadas correctas de creación o modificación y a formatear esos y cualquier otra referencia de archivo cambiado como código en línea Markdown de ruta exacta o basename único, como `out/report.html`.

#### Efecto en tokens

Un párrafo de prompt fijo siempre que este paquete esté cargado; no se añade ningún schema de herramienta, resultado de herramienta ni contexto por Turn.

#### Efecto en la KV Cache

La sección es estática en el orden 190 durante toda la vida del montaje del paquete, así que permanece en el prefijo de prompt reutilizable y no cambia entre Turnos.

## Limitaciones conocidas y trabajo pendiente

- **La coincidencia de menciones es solo por ruta exacta o basename único.** Una mención por sufijo (`out/index.html` escrito como `index.html` se resuelve; `deep/out/index.html` escrito como `out/index.html` no) permanece inerte; ampliar el comparador queda aplazado hasta que una forma real de mensaje de cierre lo necesite.
- **Los archivos creados indirectamente por comandos de terminal quedan fuera del vocabulario de coincidencia.** Nombrar ese archivo en código en línea no lo hace clicable salvo que un location de mutación correcta registre también esa ruta.
- **La entrega nativa de carpetas apunta al escritorio del Host.** Un navegador alcanzado a través de una autoridad que no es loopback omite la acción, igual que un despliegue que no reporta un abridor nativo. El reenvío SSH que hace que un Host remoto parezca local de loopback debe establecer `nativeOpen: false` en el gateway; también deben hacerlo un Host macOS/Windows headless, un despliegue WSL sin interop de Windows funcional o cualquier escritorio Linux cuya sonda de pantalla/abridor dé un falso positivo. Identificar el escritorio visible para el operador sigue siendo política de despliegue.
