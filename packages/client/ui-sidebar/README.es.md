# @deepseek-ai/dsh-client-ui-sidebar

[English](README.md) | Español

Plugin shell de la barra lateral: la fila de marca, la acción Nueva sesión, el control de colapso propiedad del layout, el asiento de región sensible al scroll y el asiento de Ajustes fijado abajo. [ui-workspace](../ui-workspace/README.es.md) es el dueño del navegador de Workspace y Session renderizado en `sidebar.workspaces`; este paquete ni deriva sus filas ni es dueño de sus preferencias de vista. El colapso al riel de 56 px propiedad del layout sigue siendo local a la presentación. Contrato: el [estándar del sistema de slots](../../../.agents/notes/implemented/architecture/2026-07-22-slot-type-chain-implementation.es.md).

La fila de marca expandida renderiza `sidebar.brand.mark` y `sidebar.brand.name` como slots únicos independientes, mientras que el riel colapsado renderiza el mismo slot de marca. Sin ocupantes, el shell usa la marca de pez y una etiqueta `DSH Local Build` que lleva la insignia `DSH_CLIENT_COMMIT_HASH` de 7 caracteres del build. Un paquete de despliegue puede sustituir cualquiera de los dos valores sin sustituir el control de Nueva sesión ni la geometría del riel; el `slots.inject()` sensible a declaraciones permite que ese paquete se active antes o después de la barra lateral.

Nueva sesión inicia el Session Intent de frontend local a la página del runtime. El runtime apunta al Workspace explícito que usa una acción con alcance; si no, al Workspace de la Session actual; si no, al Workspace activo más reciente; cuando no existe ninguno, se vacía en la página en blanco de Nueva sesión. Los controles específicos de Workspace y el selector compartido pertenecen a ui-workspace.

`SidebarRootComponentProps` compone la parte del propietario del layout, los hooks globales `useSessions` y `useWorkspaces`, la marca declarada, los slots hijos `sidebar.workspaces` y `sidebar.settings`, y los callbacks inyectados `startSession` más los de alternancia de la barra lateral. No hay store de plugin.

Durante un colapso en vivo, el shell mantiene el contenido expandido a su ancho actual mientras se desvanece durante 150 ms. Los cuatro controles superiores — la alternancia del shell y Nueva sesión más añadir y buscar renderizados a través de `sidebar.workspaces` — comparten entonces un desvanecimiento de 150 ms y una traslación de 49 px hacia la izquierda al riel de 56 px, y terminan con el deslizamiento de columna de 300 ms del layout; cada caja de control de 36 px sigue el mismo camino hasta el inset izquierdo de 10 px del riel. El control `sidebar.settings` fijado abajo comparte el ritmo del desvanecimiento pero no tiene traslación horizontal. Una página que arranca colapsada renderiza el riel estáticamente, y el modo de movimiento reducido desactiva ambas transiciones.

Las barras de scroll de la columna son una affordance de puntero: el shell reenlaza la [indirección de barras de scroll](../ui-theme/README.es.md) de ui-theme a `transparent` siempre que el puntero está fuera de ella, y mantiene dibujado el pulgar durante 2 s después de que el puntero salga, de modo que una lista a la que nadie apunta no lleva barra. La reserva que evita que las filas se muevan pertenece a la región de scroll ([ui-workspace](../ui-workspace/README.es.md)), así que revelar un pulgar nunca produce reflow.

El pie es el asiento `sidebar.settings`: la barra lateral renderiza solo el slot de layout fijado abajo y comparte su estado de columna (`wide`); ui-settings registra allí la fila de activación y el panel de ajustes.

Los exports de `/client` son el cuerpo del plugin (`apply`/`inject`) más los tipos de contrato únicamente; SidebarRoot, los componentes de fila y la derivación del árbol permanecen internos al paquete detrás del registro de slots.

## Model Experience

Ninguno: la barra lateral renderiza la lista de sesiones del navegador; nada de esto llega a una petición de modelo.

#### Efecto de KV Cache

Ninguno; este paquete ni ensambla ni envía una petición de provider.

## Limitaciones conocidas y trabajo pendiente

- **El renderizado del punto de estado de sesión es propiedad de [ui-workspace](../ui-workspace/README.es.md)** — no hay fuentes de notificación de hecho/error disponibles.
- **El comportamiento del navegador de Workspace es propiedad de la composición** — la agrupación, el orden, la búsqueda y el estado de las filas pertenecen a [ui-workspace](../ui-workspace/README.es.md), no a este shell.
- **La marca de no leído de «Nueva tarea completada» es estado de visualización local** — el tiempo de finalización > lo visto por última vez nunca llega al host.
