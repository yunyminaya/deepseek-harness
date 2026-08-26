# Agent Note: Enrutar las raíces de documentación al inicio rápido

Status: implemented

[English](2026-08-11-quickstart-documentation-home.md) | Español

## Problema

Una página de aterrizaje de documentación separada duplica el posicionamiento del producto y los resúmenes de funcionalidades de los que la página de aterrizaje del producto es dueña. Esas afirmaciones paralelas exigen sincronización y revisión sin ayudar a los lectores a llegar a las instrucciones técnicas.

## Decisión

Cada raíz de configuración regional es una página de redirección. `/` envía a los lectores a `./guide/quickstart`, y `/en/` resuelve el mismo destino relativo a `/en/guide/quickstart`. El destino relativo preserva el `DOCS_BASE` configurado cuando el sitio se aloja bajo una ruta de origen.

`docs/user/index.md` y `docs/user/index.zh.md` son dueños de la redirección como frontmatter de VitePress. El [proyector del sitio de documentación](../process/2026-07-13-documentation-site-projection.es.md) publica solo ese frontmatter para las páginas iniciales de cada configuración regional, de modo que el Markdown canónico conserva su conmutador bilingüe sin renderizar una segunda página de aterrizaje. La prueba del proyector verifica que ambas raíces regionales usan el mismo destino de inicio rápido relativo a la configuración regional.

El posicionamiento del producto y los resúmenes de funcionalidades permanecen fuera del sitio de documentación. Guía, desarrollo, referencia, búsqueda y navegación de configuración regional siguen disponibles desde la página de inicio rápido.

## Alternativas consideradas

**Conservar un hero de documentación y sincronizar su redacción.** Esto preserva una página de entrada promocional pero crea una segunda narrativa de producto cuyas afirmaciones y terminología pueden derivar de la página de aterrizaje del producto.

**Renderizar un índice de documentación en la raíz.** Un índice repite la navegación que el sitio ya proporciona e inserta otra elección antes de la primera guía accionable.

**Copiar el contenido de inicio rápido a cada raíz regional.** Dos rutas públicas serían entonces dueñas del mismo tutorial y necesitarían otro mecanismo de sincronización.

**Usar destinos de redirección absolutos al origen.** Rutas como `/guide/quickstart` ignoran `DOCS_BASE` y fallan cuando el sitio de documentación se aloja bajo una ruta de origen.

## Consecuencias

Los lectores que entran por cualquiera de las dos raíces regionales llegan de inmediato al tutorial de inicio rápido en esa configuración regional. El sitio de documentación renuncia a una superficie inicial promocional, mientras que la página de aterrizaje del producto sigue siendo la única dueña del posicionamiento y los resúmenes de funcionalidades. Las rutas raíz estables siguen siendo puntos de entrada válidos, y el contenido de inicio rápido conserva una sola fuente canónica.
