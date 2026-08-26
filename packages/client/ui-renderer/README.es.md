# @deepseek-ai/dsh-client-ui-renderer

[English](README.md) | [中文](README.zh.md) | Español

El plugin Cordis del navegador dueño de la capa de renderizado React. [`dsh-client-web`](../web/README.es.md) renderiza una página de arranque sin framework y carga el roster completo de plugins del cliente; después de que cada entrada se activa, llama a `ctx.uiRenderer.mount(container)`. Este paquete proporciona ese servicio, instala el renderizador de slots, hidrata el DOM de arranque existente, conmuta a la aplicación ensamblada antes del siguiente pintado y devuelve el disposer de desmontaje de la raíz de React.

La entrada del cliente también es dueña de la implementación React de los outlets de slots, los providers de sesión y el enlace observable-a-uSES. Los plugins de negocio pasan fuentes observables desnudas a través de los `hooks` tipados de slot; el renderizador las enlaza en el outlet. El plugin se activa después de `slots`, `sessions` y `layout`, proyecta el título de la sesión seleccionada y realiza la única llamada `renderSlot('root')` a nivel de contexto. React, React DOM, Cordis, ui-slots y ui-primitives conservan una identidad de navegador a través de la tabla de módulos estática de la carcasa web; este paquete llega como un bundle dinámico del cliente.

## Experiencia de modelo

Ninguna, ya que el renderizador de UI solo ensambla la UI del navegador y no aporta ninguna entrada visible para el modelo.

#### Efecto en KV Cache

Ninguno; este paquete no ensambla ni envía solicitudes de provider.

## Limitaciones conocidas y trabajo diferido

- **El primer frame de la aplicación espera a cada entrada del cliente** — el kernel de arranque entrega el punto de montaje solo después de que el roster del loader se asienta. La disposición por región permanece diferida.
- **El renderizado de slots no tiene integración de Suspense ni carga diferida por entrada** — el roster completo de plugins se asienta antes de que el renderizador monte la raíz.
