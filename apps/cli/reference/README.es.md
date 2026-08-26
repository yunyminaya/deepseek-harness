# Referencia de comportamiento del CLI de `dsh`

[English](README.md) | Español

Esta referencia define los modos de comando de profile, alias web, gestión de plugins y volcado de configuración. El argv se parsea una sola vez a través de [`src/args.ts`](../src/args.ts), y [`src/bin.ts`](../src/bin.ts) importa dinámicamente solo el runner seleccionado.

## Arranque del profile

`dsh --profile <name>` arranca el profile en `$DSH_HOME/profiles/<name>`. El árbol efectivo se compone sobre un root vacío aplicando, en este orden: cada parche de bundle nombrado en la lista `dsh.profile.bundles` del manifest del profile, el `cordis.patch.yml` propio del profile, el `$DSH_HOME/cordis.patch.yml` de nivel home (preferencias locales de la máquina compartidas por todos los profiles, por lo que supera a la capa del profile concreto), y cada overlay `--patch <path>` en orden de argv. Las capas posteriores ganan por fila; un parche reemplaza el valor `config` completo de la fila objetivo en lugar de fusionar claves en profundidad, y puede insertar filas nuevas. Un error de parseo, schema, resolución o arranque de plugin se informa y se sale con código distinto de cero. SIGINT y SIGTERM liberan el root montado antes de salir.

Los nombres de bundle resuelven primero desde la instalación de dsh y después desde el directorio del profile. Los bundles del paquete (`@deepseek-ai/dsh-base`, `@deepseek-ai/dsh-web-app`, `@deepseek-ai/dsh-headless`) vienen por tanto siempre de la misma instalación que el `dsh` en ejecución; los bundles fuera del árbol vienen del `node_modules` gestionado por pnpm del profile. Un `name` de plugin pelado en cualquier fila de parche resuelve mediante el recorrido de padres de Node del directorio del profile, que alcanza el fallback de instalación mantenida `$DSH_HOME/profiles/node_modules` (un symlink por paquete del que dependen la app y los bundles de la instalación, reparado en cada lanzamiento).

Los profiles `web` y `headless` se autoinicializan desde plantillas del despliegue en el primer uso (`web`: base + web-app; `headless`: base + headless). Cualquier otro profile ausente falla ruidosamente con una pista para ejecutar `dsh plugin --profile <name> add <package>`.

### Argumentos de la app

Las banderas del lanzador van primero y terminan en el primer token que no reconoce; todo lo que venga después se entrega al profile arrancado verbatim a través de `ctx.cmdlineArgs`, donde cualquier plugin de app inyectado puede parsearlo ([`dsh-cmdline`](../../../packages/boot/cmdline/README.es.md)). Así, `dsh --profile web --port 8080` llega al `--port` de la app web, `dsh --profile web --help` imprime la ayuda de esa app y no arranca nada, y `dsh --help` (sin profile al que entregárselo) imprime la del propio lanzador. `-V`/`--version` imprime la versión del lanzador cuando aparece antes del límite de argumentos de la app.

Una composición monta una sola vez. Un plugin ordinario inyecta `cmdlineArgs`, parsea los argumentos de esta app y provee como servicio lo que resolvió; cada fila configurada desde banderas inyecta ese servicio, y Loader espera por él antes de evaluar la config de la fila (`port: !!js ctx.webStartup.port ?? 3080`). Una bandera gana por tanto al valor escrito junto a ella. Esta precedencia exige que la fila conserve esa expresión; un parche de usuario que reemplace toda la `config` por literales elimina la lectura en runtime. La ayuda y los argumentos rechazados piden salir — código distinto de cero para un rechazo, 0 para la ayuda — sin activar las filas que dependen del servicio del provider. Una edición en vivo de `cordis.patch.yml` reevalúa las expresiones contra servicios que siguen activos, así que no puede restablecer un puerto servido.

Las banderas del lanzador deben ir antes de los argumentos de la app, y el parser del lanzador consume un `--`: un argumento de app que deba llegar como `--` literal necesita `-- --`. Un primer argumento de app igual a `web` o `plugin` selecciona ese subcomando en su lugar. `ctx.cmdlineArgs.get()` es una lectura inmutable compartida: varios plugins pueden parsear la misma instantánea, mientras que un profile sin lector ignora sus argumentos de app.

Las apps del despliegue son dueñas de estas líneas de comando:

| Profile | Argumentos |
|---|---|
| `web` | `--host`, `--port`, `--trusted-host` repetible, `--no-open` |
| `headless` | el texto de la tarea, como argumento posicional |

Una tarea de un solo disparo (`dsh --profile headless "run the tests"`) crea un Agent persistente nuevo a través del registro core, envía la tarea, espera a la quietud y vacía la Session antes de derivar el último texto de asistente no vacío y la razón final de `turn/end` de su intervalo duradero. Imprime el texto en stdout y sale con 0 para `completed` y 1 en caso contrario. Una invocación sin tarea es un error de uso de esa app. El profile headless del despliegue no monta ApiProxy, Host, servidor HTTP, runtime Web ni cliente de navegador; una ejecución con éxito no escribe nada en stderr ni abre ningún puerto a la escucha.

Inspecciona el árbol compuesto sin arrancarlo:

```sh
dsh --profile web --dump-default-config
dsh --profile web --patch ./extra.yml --dump-config
```

`--dump-default-config` imprime solo las capas de bundle; `--dump-config` añade el `cordis.patch.yml` del profile, el `$DSH_HOME/cordis.patch.yml` de nivel home y los overlays `--patch`. Ambos imprimen comentarios que nombran el archivo que suministró cada fila y cada overlay que la cambió; las expresiones `!!js` quedan sin evaluar, y los objetivos de parche sin emparejar se informan en stderr. Un volcado nunca ejecuta los providers de línea de comandos de la app, así que muestra el árbol compuesto antes de que se resuelva ningún argumento de app y rechaza una invocación que lleve argumentos de app.

## Gestión de plugins

`dsh plugin --profile <name> <args...>` inicializa el profile si falta (plantilla del despliegue, o solo `@deepseek-ai/dsh-base` para otros nombres), y reenvía después `<args...>` a `pnpm` con el directorio del profile como directorio de trabajo — `add`, `remove`, `why`, `update` y cualquier otro verbo de pnpm funcionan sin cambios; pnpm debe estar en PATH. Las especificaciones de ruta relativa (`.`, `../plugin` y sus formas `file:`/`link:`) se anclan primero al directorio invocante, así que `add .` desde un checkout de plugin instala ese checkout, no el profile. Tras cada ejecución con éxito, `dsh.profile.bundles` se concilia con el estado instalado: cada dependencia que resuelve a un paquete cuyo manifest declara `"dsh": { "bundle": { "patch": "./cordis.patch.yml" } }` se une a la pila de capas (así, un `update` que gane la declaración la activa), una dependencia sin bundle queda tal cual con un aviso único, y una dependencia eliminada sale de la pila.

Los providers de subagente Codex y Claude Code son Bundles opcionales separados. Añade cualquiera de los paquetes, ambos en un solo comando, o elimina cualquiera de ellos de forma independiente:

```sh
dsh plugin --profile <name> add @deepseek-ai/dsh-subagent-codex
dsh plugin --profile <name> add @deepseek-ai/dsh-subagent-claude-code
dsh plugin --profile <name> add @deepseek-ai/dsh-subagent-codex @deepseek-ai/dsh-subagent-claude-code
dsh plugin --profile <name> remove @deepseek-ai/dsh-subagent-codex
dsh plugin --profile <name> remove @deepseek-ai/dsh-subagent-claude-code
```

La operación de pnpm con éxito cambia el manifest del Profile y la lista de Bundles en disco; un Profile en ejecución conserva el conjunto de Bundles de su arranque actual. Reinicia ese Profile después de añadir, quitar o actualizar un Bundle. Esta frontera de arranque aplica a la pertenencia al Bundle, mientras que las ediciones ordinarias del `cordis.patch.yml` del Profile o de home se aplican mediante recarga en caliente. En el siguiente arranque, cada Bundle instalado registra solo su provider de Host inactivo; un Preset copiado debe habilitar aparte la fila de herramienta correspondiente para los Agents nuevos. El [README del provider Codex](../../../packages/subagent/subagent-codex/README.es.md) y el [README del provider Claude Code](../../../packages/subagent/subagent-claude-code/README.es.md) son dueños de los detalles de ejecutable, autenticación, payload y fallos; la [referencia del Bundle base](../../../packages/bundle/base/README.es.md) es dueña del cierre de dependencias por defecto.

```sh
dsh plugin --profile tui add github:deepseek-harness/turtle-ui
dsh plugin --profile tui remove turtle-ui
dsh --profile tui
```

Los plugins alojados en git que traen fuentes se compilan durante la instalación mediante su script `prepare`, que pnpm ≥10 bloquea hasta que el consumidor lo permite: el primer `add` falla con la pista `allowBuilds` de pnpm (y un puntero de dsh al `pnpm-workspace.yaml` del profile); copia allí la clave impresa y vuelve a ejecutarlo. Instalar un tarball compilado o un checkout local no necesita permiso.

## Alias web

`dsh web` es un alias fijo de `--profile web`; las banderas que le siguen pertenecen a la app web, cuyo provider de bundle ordinario las parsea. `--host` y `--port` sobrescriben los valores compuestos de las filas que los llevan, el `--trusted-host` repetible aporta autoridades de invocación a través de `ctx.webRuntime.trustedHosts` (una expresión de despliegue concatena sus propias autoridades), y `--no-open` desactiva la entrega al navegador por defecto para esta invocación. El receptor HMR del client-plugin está siempre montado y permanece inactivo hasta que un watcher separado de `pnpm run dev:web` recompila los bundles del cliente.

```sh
dsh web
dsh web --no-open
dsh web --patch ./extra.cordis.yml
dsh web --dump-config
dsh web --help
```

El runner Web de producción necesita artefactos compilados de paquete y de frontend (`pnpm run build`). Sirve `http://127.0.0.1:3080` por defecto y, en un lanzamiento local, abre esa URL host canónica solo después de que se asiente el árbol completo de Loader. Un `SSH_CONNECTION` o `SSH_TTY` heredado no vacío suprime la entrega al navegador porque el cliente SSH o el editor son dueños de la dirección reenviada local; la URL host se imprime igualmente. El CLI no soporta aún `--host 0.0.0.0` a propósito y sale con un error de uso. Inmediatamente antes de una entrega local imprime `dsh web: opening the default browser; pass --no-open to disable`; si la entrega del sistema operativo falla, un diagnóstico en stderr indica el motivo, deja el servidor corriendo y nombra la URL para uso manual. `--trusted-host` añade autoridades con nombre aceptadas por la valla de confianza del navegador de `/api`.

El apagado del proceso da al árbol de plugins hasta cinco segundos para liberarse. El primer `SIGINT`/`SIGTERM` inicia ese drenaje ordenado — `SIGTERM` es una petición de parada ordinaria de un supervisor y sale con 0 en todas las superficies, `SIGINT` informa 130; una segunda señal fuerza la salida inmediata. Si la finalización normal de un solo disparo ya está atascada en la liberación, el primer `Ctrl+C` es la escalada y sale de inmediato en lugar de ser tragado.

Todos los modos tratan el directorio invocante como el root de espacio de trabajo por defecto, cargan las instrucciones `AGENTS.md` o `CLAUDE.md` aplicables con un presupuesto de render de 65,536 bytes y usan un índice de contenido de sesión SQLite en memoria. Cada arranque de profile vigila las ediciones válidas de ambas capas de `cordis.patch.yml` (profile y home) y las vuelve a aplicar transaccionalmente; una superficie de un solo disparo sale por su apagado acotado, que libera los watchers.

Las sesiones nuevas usan por defecto el preset de permisos `workspace-write`. Las mutaciones de bash y del sistema de archivos están restringidas al espacio de trabajo de la sesión y a los roots temporales de la plataforma; las lecturas y el acceso a la red no están confinados, mientras que la visibilidad de procesos depende del backend de sandbox seleccionado — bwrap ejecuta comandos en un namespace PID privado que oculta los procesos del host, y Landlock y Seatbelt dejan la visibilidad de los procesos del host sin cambios. `DSH_PERMISSION_MODE` cambia el fallback del proceso. Los permisos de General-settings almacenados afectan a las sesiones Web posteriores, no a una ya abierta.

`DSH_TOOLS_MODE` selecciona `native`, `code` o `both` para el proceso; cualquier otro valor falla en el arranque. El agent preset `minimal` del despliegue conserva esa presentación de despliegue, fija el system prompt completo a `You are a helpful software engineer assistant.` y compone solo `bash` persistente más `str_replace_editor`. Selecciona 极简模式 al crear una sesión Web; toda otra sección de prompt y plugin orientado al modelo queda ausente de ese agent mientras el host compartido de navegador, espacio de trabajo, persistencia, sandbox y permiso permanece en su lugar.

## Comportamiento compartido del despliegue

El bundle base monta el adaptador nativo de DeepSeek, los providers de ajustes y credenciales, `web_search` estable y la telemetría de sesión deshabilitada. Las credenciales de provider resuelven desde el entorno heredado, `$DSH_HOME/.credentials.yaml`, el `.env` del directorio invocante y después `$DSH_HOME/.env`; el documento gestionado nunca se materializa en `process.env`, mientras que ambos archivos `.env` son capas de entorno de lanzamiento ordinarias. La búsqueda usa `DEEPSEEK_API_KEY` y acepta `DEEPSEEK_SEARCH_BASE_URL`; `web_fetch` está deshabilitado salvo que una capa de parche inserte un provider y lo habilite.

La telemetría de sesión permanece local por defecto. `DSH_TELEMETRY_MODE=FULL` transmite cada evento de sesión proyectado como logs OTLP/HTTP, mientras que `DSH_TELEMETRY_MODE=FEEDBACK_ONLY` sube un sufijo del log de sesión solo cuando se registra feedback. `DSH_TELEMETRY_OTLP_URL` selecciona otro colector, y cualquier `DSH_TELEMETRY_DISABLED` no vacío sigue siendo una exclusión voluntaria dura de carácter autoritativo. El base del despliegue no tiene regla de redacción de telemetría, así que las exportaciones habilitadas explícitamente pueden contener texto de mensajes, argumentos y resultados de herramientas, y rutas del espacio de trabajo; la [Agent Note default-off](../../../.agents/notes/implemented/feature/2026-08-10-telemetry-default-off.es.md) es dueña de esa decisión de despliegue.

Instala bundles de plugins externos con `dsh plugin --profile <name> add <package-or-git-spec>`. El paquete instalado es dueño de sus dependencias y aporta su capa `cordis.patch.yml` declarada. El CLI también incluye `@deepseek-ai/dsh-mcp-client` como dependencia para las capas de parche, pero ningún servidor MCP está habilitado por defecto porque cada comando de servidor es código ejecutable de confianza fuera del sandbox del agent.

## Ejecución desde el código fuente

Desde la raíz del repositorio, ejecuta `pnpm run build` por separado tras un checkout nuevo y siempre que los artefactos necesiten actualizarse, y usa después `pnpm dsh <args...>`. El script de `package.json` lanza `apps/cli/src/bin.ts` con `node --import tsx/esm` sin compilar y reenvía todos los argumentos. La falta de artefactos host de Typert hace fallar el arranque del profile con errores de resolución de módulos sin instrucción de compilación. Una vez que esos artefactos host existen, los bundles ausentes de frontend o de client-plugin fallan al arrancar con una instrucción de ejecutar `pnpm run build`. El lanzador no comprueba frescura, así que los bundles obsoletos existentes pueden ejecutar código de navegador más antiguo hasta que se recompilen. El proceso hereda el entorno de lanzamiento; pon `NODE_USE_ENV_PROXY=1` cuando una versión de Node compatible deba honrar `HTTP_PROXY` y `HTTPS_PROXY`. La forma instalada lanza el `apps/cli/lib/bin.js` compilado sin recompilar el repositorio.
