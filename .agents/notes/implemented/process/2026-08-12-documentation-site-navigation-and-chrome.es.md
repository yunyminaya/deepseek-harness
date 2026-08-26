# Agent Note: Navegación del sitio de documentación y cromado del repositorio

Status: implemented

[English](2026-08-12-documentation-site-navigation-and-chrome.md) | [中文](2026-08-12-documentation-site-navigation-and-chrome.zh.md) | Español

## Problema

La barra lateral de referencia renderizaba primero sus 43 páginas de subsistemas, por delante de cualquier otro grupo: `sectionOrder` en la configuración de VitePress no listaba posición para los grupos de subsistemas ni para el grupo que contiene la página del SDK de Python, así que `indexOf` devolvía `-1` y los ordenaba antes que las secciones ordenadas. Pulsar el ítem de navegación `参考` caía en la página de arquitectura cuya propia entrada en la barra lateral era el enlace 44 de 62, a 1549px de recorrido en una barra de 2478px — fuera del viewport. Cuatro páginas de subsistemas llevaban valores `order` ya tomados por otras páginas de la misma sección, resueltos solo por la estabilidad de `Array.prototype.sort` y el orden en que los arrays del manifest resultaron concatenarse.

La barra de navegación nombraba `/guide/` mientras el manifest publicaba la primera página de la guía en `guide/quickstart.md`, así que ese ítem servía un 404: los objetivos de navegación escritos a mano se desfasan de las rutas que el manifest publica.

Aparte, cada página canónica lleva líneas escritas para su lector de GitHub — un conmutador de idioma bajo el encabezado y, en algunas, una insignia de repositorio — que el sitio proyectaba verbatim aunque su barra de navegación ya ofrece ambos.

## Decisión

[website/docs.ts](../../../../website/docs.ts) es dueño de la ubicación de secciones. `sections` declara los grupos por locale, y `sectionSpec(locale, label)` devuelve la posición y el comportamiento de colapso de un grupo, lanzando error cuando un locale no declara ubicación para una etiqueta. Un grupo ausente de la declaración ahora hace fallar la construcción en lugar de ordenarse silenciosamente arriba. La ubicación es por locale porque las dos barras laterales nombran sus grupos de forma independiente, y una etiqueta que ambas usan — `SDK` — no puede sostener un mismo rango frente a `入门` y frente a `Guide` a la vez.

Las páginas de subsistemas se agrupan por preocupación — visión general, núcleo y ámbitos, sesiones y persistencia, modelo y contexto, ejecución y herramientas, política e interacción, plataforma y acceso — y los seis grupos temáticos se renderizan colapsados hasta que uno contiene la página leída. Los grupos ordenan al final dentro de la barra lateral de referencia: expandidos, superan en número a todos los demás grupos juntos, así que cualquier cosa situada después de ellos solo es alcanzable desplazándose más allá de la lista entera. El `order` de página deriva de la posición en el array y no de un número escrito a mano.

`landingLink(locale, collection)` deriva el destino de cada ítem de navegación a partir de `orderedPages`, el mismo orden que renderiza la barra lateral, así que un ítem siempre abre la primera página publicada de su colección.

`projectedPageContent` en [scripts/project-doc-site.ts](../../../../scripts/project-doc-site.ts) descarta la línea del conmutador de idioma y la insignia de repositorio. La coincidencia del conmutador se limita a las primeras ocho líneas para que un tutorial que muestra la convención siga renderizando su ejemplo.

El título de la barra de navegación es la marca de palabra de DeepSeek incrustada en `siteTitle`, que VitePress renderiza como HTML. Incrustar es lo que deja que los rellenos de `currentColor` de la marca sigan el tema activo; `themeConfig.logo` renderiza un `<img>`, que congela la marca en los colores que su archivo declara y necesitaría un recurso por tema. La barra de desplazamiento de la barra lateral permanece invisible y aparece al desplazarse, marcada con un atributo `data-` y no con una clase porque Vue reescribe `class` íntegramente cuando parchea el elemento.

## Alternativas consideradas

**Un tokenizador de búsqueda para consultas en chino.** Construido y revertido. La premisa — que MiniSearch deja la prosa china como frases enteras intokenizables — se probó contra un término (`子代理`) que no aparece en ningún lado del corpus; las páginas chinas escriben `Subagent` y `子 agent`. Medido contra el índice sin modificar, `插件配置` devuelve 120 aciertos, `会话持久化` 85, `工作流` 28, `沙箱` 12, y cada uno clasifica su propia página primero: `prefix: true` ya alcanza los términos chinos a través de los tokens cortos que produce la puntuación. Los pares de caracteres adyacentes engordaron el índice chino de 1,23MB a 2,12MB sin ganancia. El intento también sacó a la luz una trampa que conviene conservar: VitePress envía las funciones de opciones de búsqueda al navegador a través de `Function.prototype.toString` y las reconstruye con `new Function`, así que cualquier función así que cierre sobre una constante a nivel de módulo lanza error en un ámbito vacío y devuelve silenciosamente cero resultados.

**Colocar los grupos de subsistemas directamente tras `概念`.** Rechazado: restaura la página de arquitectura arriba pero deja la referencia generada, la API de Cordis y el cookbook por debajo de las 43 filas.

**Reescribir el texto de enlaces de nombre de archivo durante la proyección.** La tabla índice de subsistemas escribe `[core.md](core.md)`, que en el sitio se lee como índice de archivos del repositorio. `scripts/project-doc-site.spec.ts` afirma ese formato exacto de fila, así que los nombres de archivo son una convención deliberada y no un descuido; cambiar lo que el sitio muestra significa cambiar la convención y su puerta juntas, no esquivarlas en el proyector.

## Consecuencias

La barra lateral de referencia mide 1452px con todos los grupos de subsistemas colapsados, contra 2478px antes, y la página de arquitectura es su primera entrada. La ubicación y el colapso de secciones se declaran en un solo manifest en vez de repartirse entre el manifest y la configuración, y `scripts/project-doc-site.spec.ts` fija tres invariantes: toda página con barra lateral resuelve una ubicación, una sección sin declarar se rechaza y dos páginas no comparten `order` dentro de una sección.

El Markdown canónico queda intacto tras despojar el cromado — el conmutador y la insignia siguen sirviendo a los lectores de GitHub. El costo es que el proyector ahora conoce dos convenciones de presentación del corpus fuente, y una página escrita con otra redacción de conmutador no coincidiría.

La marca de palabra es una segunda copia de una marca que también vive en `apps/web/public/favicon.svg` y `packages/client/ui-primitives/src/FishLogo.tsx`, cada una con su propia presentación. Un cambio en la marca de palabra de DeepSeek llega al sitio de documentación solo actualizando esta copia.
