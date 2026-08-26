# @deepseek-ai/dsh-client-ui-permission-presets

[English](README.md) | [中文](README.zh.md) | Español

Superficies de permiso en el navegador para dos vidas distintas. La fila de Ajustes generales lee el descriptor de Settings `permission` expuesto explícitamente, deriva sus opciones del enum dinámico `defaultPreset` del host y escribe una operación de ruta `settings.mutate` con la revisión del descriptor. Su observable viaja en el compartimento `hooks` del sistema de slots, así que el renderizador es dueño del enlace de hooks de React; una invalidación por push vuelve a obtener el descriptor. Este valor se aplica solo cuando se crea una sesión posterior; cambiarlo no conmuta la sesión actual. Elegir Full access requiere un reconocimiento explícito de riesgo antes de que la fila lo escriba.

La superficie de la sesión actual sigue siendo una DECORACIÓN popupSelect colgada del comando `/permission` del host (`ctx.commandUi.decorate`). Una decoración no es un segundo comando: el comando del host conserva su fila en el menú de barra, la ruta argumentada (`/permission <preset>` conmuta directamente) y el registro de ciclo de vida duradero; la decoración reemplaza solo la invocación desnuda por el selector: una lista plana de presets con el valor actual marcado como activo y nombres de preset en kebab-case renderizados como etiquetas en title-case (`workspace-write` → `Workspace Write`, gemelo de la transformación de visualización del chip del compositor), donde una elección envía la línea de comando `/permission <preset>`. Las opciones y la marca de activo leen la proyección `permissions` de la sesión (el mismo select calculado por el host que renderiza el chip del compositor), así que ambas superficies de la sesión actual comparten una fuente de lectura y una ruta de escritura, y el frame de proyección enviado es la única confirmación que ambas siguen. La decoración está disponible exactamente mientras la clave de proyección está presente; una composición sin permisos no muestra ni selector ni fila de Ajustes.

Los exports de `/client` son el cuerpo del plugin (`apply`/`inject`).

## Experiencia de modelo

Indirectamente, a través de los hechos de permiso que escriben sus dos superficies: la fila de Ajustes hace que una sesión futura comience con eventos de perilla de valor completo (`permission/preset`, `sandbox/mode`, `approval/policy`), mientras que el selector `/permission` añade los mismos hechos cuando conmuta la sesión actual; esos eventos seleccionan el modo de sandbox y la política de aprobación que resuelven las llamadas de herramienta posteriores, y la interacción con el selector no añade contenido al prompt.

#### Efecto en KV Cache

Sin invalidación directa; los consumidores de la perilla son dueños de cualquier cambio en el prefijo de la solicitud.

## Limitaciones conocidas y trabajo diferido

- **La fila de Ajustes es solo Web** — los clientes no Web pueden seguir conmutando la sesión actual a través de `/permission`, pero no reciben esta contribución del navegador.
