# Agent Note: Las apps son dueñas de su línea de comandos a través de `ctx.cmdlineArgs`

Status: implemented

[English](2026-08-06-app-owned-command-line.md) | Español

## Problema

Tras los profiles, las composiciones se podían instalar, pero sus líneas de comandos no. `apps/cli` seguía declarando la familia de flags Web (`--host`, `--port`, `--dev`, `--workspace-root`, `--trusted-host`) y el posicional de tarea de un solo uso, y luego derivaba patches para los id de fila que tenía codificados (`webserver`, `api-gateway`, `connection`, `web-runtime`). Una app fuera del árbol, como [turtle-ui](https://github.com/deepseek-harness/turtle-ui), podía contribuir filas pero no tenía forma de aceptar un flag: `dsh --profile tui --resume <session>` no tenía dónde analizarse, y `dsh --profile web --help` imprimía la ayuda del launcher en lugar de la de la web app.

## Decisión

El launcher analiza solo lo que le pertenece — `--profile`, `--patch`, los volcados de configuración — y entrega **todo lo que viene después de sus propios flags** al árbol arrancado, verbatim. La división es posicional: el primer token que el launcher no reconoce inicia los argumentos de la app (el `passThroughOptions` + `allowUnknownOption` + `helpOption(false)` de commander). Un `dsh -h` desnudo, que no tiene ninguna app a la que entregar el flag, sigue imprimiendo la ayuda del propio launcher.

El paquete nuevo `@deepseek-ai/dsh-cmdline` es dueño de la entrega. Un launcher llama a `provideCmdline(ctx, host)` antes de que se monte ninguna entrada, proporcionando `ctx.cmdlineArgs` (cuya interfaz completa es `get(): readonly string[]`) y `ctx.appExit`. Cualquier plugin de app ordinario puede inyectar `cmdlineArgs`, llamar a `parseCmdline(ctx, program)` con su propio programa de commander y proporcionar el valor resuelto como un servicio propiedad de la app desde la acción del programa. Su fila de Loader no lleva ninguna marca de launcher ni tipo especial, y el launcher no inspecciona la composición en busca de un dueño. Varios plugins pueden leer la misma instantánea inmutable; un profile sin lector ignora sus argumentos de app. Las filas configuradas desde un provider inyectan su servicio y leen expresiones de configuración perezosas directas (`port: !!js ctx.webStartup.port ?? 3080`), de modo que un flag vence al valor escrito junto a él y no se escribe nada de vuelta en ninguna fila.

El arranque monta la composición una sola vez. Cordis retiene cada fila hasta que sus inyecciones están activas; el Loader interpola entonces el `!!js` de esa fila contra el contexto de plugin listo para la inyección, inmediatamente antes de la activación. Include mantiene las expresiones de filas anidadas crudas hasta que su fila objetivo llega a este punto. `--help` deja ausente el servicio del provider, de modo que las filas dependientes nunca se activan, y una recarga de patch en vivo interpola de nuevo contra el servicio que sigue activo, de modo que un puerto servido no puede restablecerse en silencio.

Las apps incluidas trasladaron sus flags a sus bundles: `dsh-web-app` es dueña de la familia Web, y `dsh-headless` es dueño del posicional de tarea y rechaza una tarea ausente como error de uso. `apps/cli/src/web.ts` ya no existe; `runProfile` ya no conoce ningún id de fila objetivo de flags. Fuera del árbol, turtle-ui ganó `--resume <session>` / `--session <id>` de la misma manera, que es la validación real del diseño: un plugin instalado añadió un flag sin ningún cambio en el launcher.

Dos consecuencias más. El Loader monta las filas hermanas de forma concurrente, de modo que una fila puede activarse mientras otra sigue montándose o mientras todo el arranque está haciendo rollback; por eso el bundle Web publica su URL solo después de que su propio árbol de Loader se asiente. El plugin de runtime del bundle Web es dueño también de la sección de prompt de la fuente del harness, de modo que `dsh web` y `dsh --profile web` arrancan de forma idéntica sin configuración de launcher específica de Web.

## Por qué el Loader es dueño del orden

Tres hechos del framework dan forma al mecanismo:

- **Las filas de un profile llegan dentro de la opción `patches` del include raíz.** Include declara el marcador portador de árbol `EntryGroup.key` (igual que Group), de modo que el Loader mantiene su config — las listas de entradas y patches, incluido el propio `path` de Include — literal en lugar de evaluar recursivamente los nodos `!!js` anidados en el contexto del Include; cada expresión se resuelve en la fibra de su fila objetivo.
- **Cordis activa una fibra solo después de que todas las inyecciones declaradas estén activas.** Inmediatamente antes de cada activación, Cordis ejecuta el waterfall `internal/config` contra el contexto propio de la fibra; el listener del Loader interpola la config cruda después de que Cordis toma una instantánea de sus servicios inyectados.
- **El reemplazo de provider y el HMR deben preservar el mismo contrato.** La reactivación de una fibra vuelve a ejecutar el waterfall, el HMR lleva la config cruda a la fibra de reemplazo, y una fila pendiente acepta cambios de opciones sin evaluar prematuramente expresiones contra servicios ausentes.

Esto deja el orden de dependencias en la activación de Cordis y en la interpolación del Loader, que son sus dueños. Las filas conservan su `inject` y su config, el Loader monta la composición una sola vez, y el launcher solo proporciona los servicios de argv y de ciclo de vida del proceso.

## Alternativas consideradas

- **Escribir los valores resueltos en cada fila** (una actualización de config por fila, más una capa de patch devuelta al launcher para que una recarga no pudiera deshacerla): funcionaba, pero suponía patches viajando de una app al launcher y de vuelta, dos mecanismos para un mismo hecho, y un reciclaje cuya corrección dependía de los internals de reinicio del Loader. El mantenedor rechazó el viaje de ida y vuelta; el servicio que leen las filas lo reemplazó por completo.
- **Liberar filas limpiando su `inject`**: funcionaba de forma aislada y fallaba en el árbol web real, porque limpiar `inject` es exactamente lo que pierde las inyecciones estáticas del plugin. El fallo es silencioso hasta que un plugin lee un servicio que declaró.
- **Montaje de dos pasadas gestionado por el launcher**: puede activar un provider antes de que se apliquen los lectores, pero duplica la composición, convierte el orden en una preocupación del launcher y oculta el defecto del Loader de que las expresiones anidadas se evaluaban en el contexto del include en lugar del contexto inyectado de la fila objetivo.
- **Que el launcher ejecute la función de comando de cada bundle antes del arranque** (sin intervención de Cordis): estrictamente anterior a «arrancar, luego help», pero convierte el arranque de la app en un segundo protocolo de plugins fuera del árbol. Un provider ordinario inyectado con `cmdlineArgs` mantiene un único protocolo y sigue siendo volcable (dumpable) y parcheable (patchable).
- **Un dueño de línea de comandos impuesto por el launcher**: rechazar cero o múltiples lectores arbitraría solapamientos como `-h`, pero `get()` es una lectura inmutable y una composición normal puede necesitar varios servicios propiedad de la app. Los plugins comparten por tanto la instantánea y gestionan cualquier interacción con el parser mediante la composición ordinaria.
- **`instanceof CommanderError`**: un plugin fuera del árbol trae su propia copia de commander, de modo que la identidad de la clase difiere y un `--help` impreso se relanzaba como un fallo de carga fatal. Los errores de control de flujo de Commander se detectan estructuralmente en su lugar.

## Consecuencias

- Los flags, el texto de ayuda y los errores de uso de una app viven con las filas que configuran; añadir un flag a un plugin instalado no requiere ningún cambio en el launcher.
- El launcher no reconoce ninguna fila de app: la fila de telemetría sigue siendo su única sonda de composición (para el cambio de entorno), SIGTERM sale con 0 en todas las superficies, cada arranque vigila sus capas de patch de usuario, y el runner de un solo uso sale a través de `ctx.appExit` como cualquier otra app.
- `--help` deja pendiente toda fila que dependa del servicio del provider y solicita una salida acotada; las filas no relacionadas pueden activarse de forma concurrente antes del teardown.
- Un servicio propiedad de la app no tiene ningún provider declarado estáticamente: un bundle que distribuya filas de Consumer sin ese provider falla en el settlement (asentamiento) con entradas pendientes que nombran el servicio, no en la carga.
- Un patch de usuario que reemplaza el `config` completo de una fila elimina sus expresiones y, con ellas, la precedencia del flag para esa fila.
- Los flags del launcher deben preceder a los argumentos de la app; un primer argumento de app igual a `web` o `plugin` selecciona ese subcomando en su lugar, `-V`/`--version` sigue perteneciendo al launcher antes de esa frontera, y el parser del launcher consume un `--`, de modo que un `--` literal para la app necesita `-- --`.
- `--dump-config` nunca ejecuta los providers de línea de comandos de la app, de modo que imprime la composición antes de que se resuelva cualquier argumento de app y rechaza una invocación que lleve argumentos de app.
