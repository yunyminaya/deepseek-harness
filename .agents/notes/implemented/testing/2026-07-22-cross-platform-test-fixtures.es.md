# Agent Note: Mantén semánticos los tests de plataformas soportadas

Status: implemented

[English](2026-07-22-cross-platform-test-fixtures.md) | Español

## Problema

Las suites unitarias y de cobertura corren en Windows, macOS y Linux, pero un comportamiento neutral de plataforma puede quedar oculto detrás de un fixture específico de plataforma. Las rutas POSIX literales se vuelven rutas relativas a unidad en Windows, una URI `file:` alojada puede ser una ruta UNC válida allí, y el cierre de pipes de hijo o la programación del event loop no se asienta en el mismo punto en cada host. Los estados de sistema de archivos solo-POSIX como FIFOs, bits de modo ejecutable y bits de búsqueda de directorio no tienen fixture directo en Windows.

Tratar la sintaxis de los fixtures como comportamiento de producto reporta o bien regresiones falsas, o bien fomenta una normalización de producción que borra la semántica nativa de las rutas.

## Decisión

Los tests de comportamiento neutral de plataforma construyen rutas absolutas y URIs `file:` con las APIs `node:path` y `node:url` del host, y luego afirman salida absoluta nativa o salida estable relativa al workspace según lo exija el contrato. Los fixtures de URI inválida usan codificaciones rechazadas por `fileURLToPath()` en todas las plataformas soportadas.

Los tests de fallo de transporte inyectan el escritor de mensajes de la conexión y entregan el mismo error asíncrono de callback de escritura que reportaría un stream real de Node. El escritor de producción sigue escribiendo mensajes enmarcados al stdin del hijo. Esto mantiene vivo a un hijo real mientras el test distingue determinísticamente el fallo de transporte del exit del proceso sin meterse en los handles de pipe específicos de plataforma.

El teardown del language server apunta a todo el árbol descendiente mediante un id de grupo de procesos negativo en POSIX y `taskkill /T /F` síncrono en Windows. Windows suprime solo el estado de árbol-ya-ausente de taskkill; los fallos de comando, permiso y otros fallos de kill de árbol siguen siendo fallos de teardown. Una consulta de provider de solo lectura reintenta una sola vez solo cuando su transporte agrupado seleccionado falla antes o durante esa consulta; los errores de un servidor aún vivo no se reproducen. Los tests de terminal esperan su salida renderizada observable en lugar de asumir que un turno de event loop es suficiente.

Los tests de un primitivo genuinamente solo-POSIX usan una exclusión estrecha de Windows en ese caso. Los casos cross-platform adyacentes siguen fijando el rechazo de archivos no regulares, el rechazo de comandos no disponibles y el rechazo de directorios de trabajo inaccesibles. Las rutas Windows soportadas permanecen dentro de la puerta de cobertura por archivo en lugar de excluirse junto con sus archivos de test.

## Alternativas consideradas

**Normalizar todas las rutas y URIs a cadenas POSIX.** Esto uniformaría las aserciones pero cambiaría el comportamiento correcto de Windows: las rutas externas son rutas absolutas nativas, las URIs de archivo UNC son válidas y los homes configurados se resuelven a través de las reglas de rutas del host.

**Manipular los internals de los pipes de hijo hasta que una escritura falle.** Los descriptores CRT y los handles libuv tienen titularidad distinta entre hosts y versiones de Node, así que esto probaría maquinaria de fixture no documentada en lugar del contrato de fallo de escritura de la conexión.

**Omitir archivos o paquetes enteros en Windows.** Las exclusiones amplias ocultarían comportamiento soportado. Solo se excluye el fixture individual cuyo estado no puede existir en Windows; el contrato circundante permanece cubierto.

## Consecuencias

Los fixtures portables son ligeramente más explícitos porque las rutas esperadas se derivan de constantes nativas compartidas y los fallos de transporte entran por un hook de escritor estrecho. Las exclusiones solo-de-plataforma requieren una aserción cross-platform vecina para el comportamiento de producto que soportan. El teardown de Windows depende del comando `taskkill` del host después de que el cierre de protocolo gracioso haya fallado; un resultado síncrono exitoso mantiene acotado el dispose y hace observable el exit del descendiente antes de que la limpieza devuelva, mientras que un kill de árbol fallido permanece visible para el disposer.
