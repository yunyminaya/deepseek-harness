# dsh-home-paths

[English](README.md) | Español

Helpers de rutas de sistema de archivos compartidos para los datos de usuario de DeepSeek Harness.

## Home de DSH

`resolveDshHome()` resuelve el home de DeepSeek Harness de raíz única. Precedencia, de mayor a menor: una ruta configurada explícita, `$DSH_HOME` y luego `~/.dsh`. El harness mantiene todos los datos de usuario bajo una sola raíz.

`dshHomePath(...segments)` une segmentos hijos a ese home resuelto con las reglas de ruta de la plataforma de Node. Sin segmentos, devuelve el propio home.

`dshHomeDisplay()` nombra simbólicamente una raíz activa para las rutas visibles al usuario: `~/.dsh` para el home por defecto, `$DSH_HOME` para cualquier home configurado. Nunca filtra una ruta absoluta de la máquina.

`DSH_HOME_DIR_NAME` es dueño del nombre por defecto del directorio de datos de usuario: `.dsh`.

`defaultDshHome()` devuelve el home por defecto de DeepSeek Harness uniendo el directorio home del sistema operativo con `.dsh`, usando las reglas de ruta de la plataforma de Node.

`expandHomePath()` expande los prefijos `~`, `~/...` y `~\\...` estilo Windows contra el directorio home del sistema operativo. Deja intactas las rutas sin tilde y las de `~user/...`.

## Rutas de watch

`canonicalizeWatchPath()` da a un watcher nativo del sistema de archivos una grafía estable de su objetivo. Resuelve el ancestro existente más profundo mediante `fs.realpath()` y restaura cualquier sufijo ausente, de modo que un archivo o directorio puede seguir vigilándose antes de crearse. En particular, los alias 8.3 de Windows no pueden mezclarse con las rutas largas que emite el backend nativo del watcher.

Este paquete es deliberadamente pequeño y libre de dependencias del harness para que los paquetes de producto puedan compartir las convenciones de rutas de datos de usuario sin depender unos de otros.

## Limitaciones conocidas y trabajo diferido

- **La expansión es deliberadamente estrecha** — solo el `~` desnudo, `~/...` y `~\\...` usan el home actual del sistema operativo; las formas de usuario con nombre como `~alice/...`, las variables de entorno y las expresiones de shell permanecen sin cambios.
- **La canonicalización lee pero nunca muta** — `canonicalizeWatchPath()` realiza sondeos de `realpath` y propaga los errores distintos de la ausencia; los llamadores siguen siendo dueños de la creación de directorios, los permisos y la política de confianza de la ruta resultante.
