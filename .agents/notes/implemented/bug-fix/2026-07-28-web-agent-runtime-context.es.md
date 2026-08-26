# Agent Note: Los agentes Web reciben contexto de runtime explícito

Status: implemented

English | [中文](2026-07-28-web-agent-runtime-context.zh.md)

## Problema

La base CLI compartida configuraba una persona de despliegue vacía, el overlay Web no la reemplazaba, y el launcher Web no añadía ninguna sección de fuente o superficie de interacción. Un header de sesión registraba su directorio de trabajo para las herramientas y la persistencia, pero el prompt del modelo no enunciaba ese directorio ni identificaba la DeepSeek Harness Web GUI. Una petición como «cambia el tema de esta página» hacía por tanto que el agente buscara en el proyecto seleccionado una página sin especificar, incluso cuando el usuario se refería a la GUI que ejecuta la sesión.

## Decisión

El perfil Web compone los bundles `dsh-base` y `dsh-web-app`. El bundle Web aporta una persona concisa de agente-de-código con el `{{model}}` resuelto y el `{{cwd}}` de la sesión; su plugin `web-runtime` añade la sección `app:web-surface` cuando `surfaceContext` es true. Antes de montar el árbol de perfiles, el alias `dsh web` lee esa misma configuración compuesta e instala la sección `harness:source` existente solo cuando el contexto de superficie está habilitado. El bundle headless y un perfil que posee su prompt completo ponen `surfaceContext: false`, suprimiendo el prompt Web y los hechos de shell gestionados; el alias Web también suprime la sección de fuente sin comprobar una ruta de overlay. Cada contribución de prompt montada sigue activándose antes de que consumidores como el bucle del agente puedan emitir un header de petición. La [decisión source-checkout/workdir](2026-07-30-source-checkout-workdir-distinction.es.md) posee la redacción de la sección de fuente y su advertencia de no inferir una ruta desde la otra.

La sección Web trata las referencias sin calificar a «esta página», «esta GUI» o «esta app» como referencias a la DeepSeek Harness Web GUI. También enuncia que el browser no aporta contexto implícito de DOM, rutas ni capturas, de modo que el modelo pueda identificar el producto sin reclamar estado visual que no recibió. El texto ensamblado se registra en `request/header`, preservando el invariante modelo-visible/registrado.

## Verificación

Los tests unitarios del runtime Web fijan el comportamiento de `surfaceContext` habilitado y deshabilitado, mientras que el test unitario del alias Web fija el gating de la sección de fuente por defecto-activado y explícitamente-apagado desde la fila compuesta. El escenario Web keyless fresh-round-trip arranca el base shipped más el bundle Web, corre una sesión real a través de la aplicación HTTP/SSE, y hace snapshot del prefijo del system prompt con las rutas de fuente y directorio de trabajo normalizadas. El snapshot fija la identidad del harness, el checkout de fuente, la orientación Web y la persona resuelta de agente-de-código en orden de petición. El snapshot Core Web aplica el overlay RL y fija su system prompt completo sin sección de fuente ni Web.

## Alternativas consideradas

**Enviar URL, DOM o capturas con cada prompt.** El fallo observado necesitaba orientación estable del producto, mientras que la URL raíz actual no identifica un componente seleccionado y no existe captura visual en el contrato de mensajes. Añadir estado dinámico de página exigiría un diseño separado de entrada-de-modelo registrada y no está implicado por este fix.

**Exigir que el Workspace de la sesión sea el checkout del harness.** El cwd del Workspace es el objetivo de tarea del usuario y puede legítimamente ser un proyecto vacío u otro repositorio. Confluirlo con la ubicación de fuente de la aplicación rompería ese límite y dejaría ambiguas las sesiones instaladas o lanzadas externamente.

**Poner la redacción Web en la identidad global del harness.** `dsh-system-prompt` sirve a TUI, ACP, SDK y despliegues custom que no corren en un browser. La app Web que compone posee este hecho de superficie.

**Cambiar la sección de ubicación de fuente existente para cada superficie CLI.** La sección de fuente se comparte con TUI y enuncia solo el hecho del checkout. Mantener la orientación Web separada preserva ese contrato reutilizable y evita decirle a agentes headless o de terminal que están en un browser.

## Consecuencias

Las peticiones Web ordinarias ganan un prefijo de prompt corto y estable y pueden invalidar una vez las cachés de prefijo del proveedor cuando se despliegue este cambio. Los agentes pueden distinguir el checkout de fuente de la GUI del Workspace seleccionado y resolver las referencias ordinarias a la app actual sin un round trip de aclaración. Las referencias a un estado visual específico permanecen acotadas por la declaración explícita sin-DOM/sin-ruta/sin-captura y pueden seguir exigiendo una ruta, descripción o adjunto. Los perfiles de prompt-completo pueden salirse vía la configuración de composición del runtime Web sin una comprobación de ruta del launcher.
