# Agent Note: Resuelve las rutas del sistema de archivos contra el cwd de la sesión del llamador

Status: implemented

[English](2026-07-02-fs-per-session-cwd.md) | Español

## Problema

El puente ACP (Agent Client Protocol) da a cada sesión su propio workspace: `session/new` registra el directorio del proyecto del cliente de automatización como `SessionHeader.cwd`, y `dsh-tool-bash` fija por defecto el `workdir` de cada llamada de bash al `session.header.cwd` del agent llamador (véase [el paquete ACP](../../../../packages/acp/acp) y `resolveWorkdir` en `dsh-tool-bash`). Así, un comando de bash en la sesión A se ejecuta en el proyecto de A, y en la sesión B en el de B — un proceso de servidor, N workspaces.

La resolución del sistema de archivos usaba un cwd de carga de plugin mientras que bash usaba el directorio del proyecto de la sesión. Las rutas relativas por tanto discrepaban siempre que el proyecto del cliente de automatización difería del directorio de lanzamiento del servidor; las instantáneas ocultaban el bug haciendo esas rutas idénticas.

Un cwd absoluto válido puede tener por sí mismo dos padres aparentes: cuando contiene `symlink/..`, la búsqueda del sistema de archivos sigue el symlink antes de aplicar `..`, mientras que `path.resolve()` borra ambos componentes léxicamente. Resolver la política de sandbox léxicamente mientras se lanzaba bash desde el cwd crudo otorgaba el padre léxico no relacionado, denegaba escrituras en el workspace real y dejaba que las herramientas de sistema de archivos resolvieran rutas relativas hacia el directorio equivocado.

Un cwd symlink ordinario expone la misma distinción cuando la ruta relativa solicitada contiene `..`: un proceso atraviesa desde el objetivo físico del symlink, mientras que `path.resolve(cwd, path)` atraviesa desde su grafía léxica. Las lecturas seleccionarían por tanto un archivo distinto del que seleccionarían bash o una mutación en sandbox para la misma ruta suministrada por el modelo.

## Decisión

Pasa el cwd de la sesión del llamador a la resolución de rutas, exactamente como `dsh-tool-bash` ya hace con `workdir`. Cuando el cwd o la ruta solicitada contienen un segmento de padre, resuelve el cwd a su identidad nativa del sistema de archivos antes de cualquier unión léxica; las grafías ordinarias de cwd permanecen estables para la visualización cuando ningún cruce hace observable su identidad. Reutiliza la raíz resuelta de la política de sandbox para las mutaciones y las llamadas de bash en sandbox, de modo que una llamada tiene una única identidad de workspace. El **llamador** (la herramienta) aporta el cwd; el provider no lee una sesión ni un agent.

- `FileSystem.resolve` acepta `resolve(path: string, opts?: { cwd?: string; signal?: AbortSignal }): Promise<FsTarget>`. `opts.cwd` es la base contra la que se resuelve una `path` RELATIVA; una `path` absoluta lo ignora; omitir `opts.cwd` usa el valor por defecto propio del backend. `opts.signal` cancela la resolución cuando el backend realiza I/O. El objeto de opciones mantiene juntos ambos controles de resolución propiedad del llamador sin crecimiento posicional.
- `dsh-fs-local.resolve` usa `resolveLocalTarget(opts?.cwd ?? this.config.cwd, path)`. `config.cwd` sigue siendo el valor por defecto para un llamador que no aporta cwd de sesión.
- Los `read`/`write`/`edit` de `dsh-tool-fs` derivan el cwd de sesión a través de un helper compartido `sessionCwd(exec, requestedPath)` (`exec.agent?.session.header.cwd`, en espejo de `resolveWorkdir` de bash) y lo pasan a `resolve`. El helper usa semántica nativa de realpath cuando un segmento de padre en cualquiera de los valores podría cruzar un symlink, conservando en caso contrario las grafías ordinarias; una mutación en sandbox reutiliza el `workspaceRoot` de la política completa; un llamador que no es agent / sin header produce `undefined`, así que el backend aplica su valor por defecto.

## Alternativas consideradas

### Por qué el llamador aporta el cwd (no el provider)

El contrato del provider no debe depender de `dsh-agent` / `dsh-session` — es un backend de almacenamiento de texto que también satisface una implementación en sandbox o remota, y esas no tienen noción de «sesión de agent». La herramienta ya recibe el `ToolExecution` (`exec`), que transporta el agent, así que la herramienta es el lugar adecuado para proyectar `exec → cwd` y entregar al provider una cadena simple. Es la convención de «explícito > implícito en los límites de paquetes»: el directorio base llega como argumento explícito sobre el que actúa el provider, no de contrabando haciendo que el provider hurga en una sesión que no debería conocer. También coincide uno a uno con `dsh-tool-bash`, así que las dos superficies de archivos orientadas al modelo resuelven rutas de forma idéntica.

El valor por defecto vive en UN solo lugar — el `config.cwd` del provider. `sessionCwd` retorna `undefined` en lugar de `process.cwd()` cuando no hay sesión, así que la herramienta nunca fabrica una base que el provider elegiría por su cuenta.

## Consecuencias

- En la demo ACP, las herramientas de fs y bash coinciden en el workspace de cada sesión; un cliente de automatización puede seleccionar cualquier directorio de proyecto absoluto y ambas familias de herramientas actúan sobre él.
- Un cwd de sesión que contenga `symlink/..`, o un cwd symlink ordinario emparejado con una ruta relativa que cruce el padre, se resuelve desde el mismo workspace físico para bash, las herramientas de sistema de archivos y la concesión de sandbox; el padre léxico no recibe ninguna concesión.
- Sin cambios en la identidad de `FsTarget`: `targetKey` sigue siendo el realpath de la ruta absoluta resuelta, así que el encadenamiento del estado observado y la identidad de symlink no se ven afectados — un cwd por sesión correcto produce la misma clave a la que apunta bash.
- Compatible hacia atrás: toda llamada existente `resolve(path)` (todas en tests) sigue funcionando; el argumento nuevo es opcional.
- La demo stdio de sesión única no se ve afectada: no aporta cwd de sesión (la sesión de su agent no tiene `cwd`), así que la resolución cae en `config.cwd = process.cwd()`, que es el workspace.
