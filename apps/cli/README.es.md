# `@deepseek-ai/dsh`

[English](README.md) | Español

El comando `dsh` es el lanzador de producto para los perfiles: pilas ordenadas de capas de patch de plugin-bundles bajo los overrides propios del usuario. [`src/args.ts`](src/args.ts) es dueño de la gramática del comando, y [`src/bin.ts`](src/bin.ts) carga solo el runner seleccionado. Los comandos inválidos, las opciones de otro modo, los errores de configuración y los fallos de arranque salen con código distinto de cero.

## Modos de entrada

| Comando | Propósito |
|---|---|
| `dsh --profile <name>` | Arranca el perfil nombrado bajo `$DSH_HOME/profiles/<name>`. |
| `dsh --profile headless "job"` | Ejecuta una sesión nueva persistida, imprime la respuesta final y sale. |
| `dsh web` | Alias de `--profile web`. |
| `dsh plugin --profile <name> <pnpm args>` | Gestiona los plugins de un perfil reenviando a pnpm en el directorio del perfil. |

El directorio desde el que se invoca es la raíz de workspace por defecto. Los perfiles `web` y `headless` se auto-inicializan en el primer uso a partir de plantillas distribuidas; cualquier otro perfil debe crearse mediante `dsh plugin`.

## Argumentos de la aplicación

El lanzador analiza solo sus propias banderas y entrega todo lo que viene después al perfil arrancado, donde cualquier plugin de aplicación inyectado puede analizar la instantánea inmutable compartida ([`dsh-cmdline`](../../packages/boot/cmdline/README.es.md)). Por tanto, las banderas del lanzador van primero, y el primer token que el lanzador no reconoce inicia los argumentos de la aplicación:

```sh
dsh --profile web --port 8080       # --port belongs to the web app
dsh --profile tui --resume <id>     # example, assuming the tui profile is installed; --resume belongs to the terminal app
dsh --profile headless "run the tests"
dsh --profile web --help            # the web app's flags, not the launcher's
dsh --help                          # the launcher's own help
```

## Perfiles

Un directorio de perfil contiene un `package.json` (dependencias de plugins fuera del árbol más el manifest del perfil `dsh.profile` con su lista ordenada `bundles`) y un `cordis.patch.yml` (la capa de patch propia del usuario).

El árbol se compone sobre una raíz vacía:
- el patch de cada bundle en el orden de `dsh.profile.bundles`
- luego el `cordis.patch.yml` del perfil, y después el `$DSH_HOME/cordis.patch.yml` del nivel de home
- y después los overlays `--patch`

Los bundles nombrados en `dsh.profile.bundles` se resuelven primero desde la instalación de dsh (`@deepseek-ai/dsh-base`, `@deepseek-ai/dsh-web-app`, `@deepseek-ai/dsh-headless`) y luego desde el `node_modules` propio del perfil, donde pnpm instala los plugins fuera del árbol.

Usa `--dump-default-config` y `--dump-config` para inspeccionar el árbol compuesto sin arrancarlo.

La [referencia de comportamiento del CLI](reference/README.md) es dueña de la precedencia exacta de capas, las banderas, el comportamiento de apagado, los valores por defecto de despliegue y la ejecución desde el código fuente.

## Desarrollo

Las ejecuciones de producción requieren el paquete compilado y los artefactos de frontend. Desde la raíz del repositorio, ejecuta `pnpm run build` por separado y luego usa `pnpm dsh <args...>` para ejecutar la entrada TypeScript y reenviar todos los argumentos; la [referencia de ejecución desde el código fuente](reference/README.md#source-execution) es dueña del contrato de resolución de módulos.
