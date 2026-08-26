# Agent Note: Un solo orden para las fuentes de configuración, y lo que un archivo descubierto no puede decidir

[English](2026-08-04-configuration-source-ownership.md) | Español

Status: implemented

## Problema

`$DSH_HOME/.env` acababa de [convertirse en una capa de entorno ordinaria](2026-08-04-credentials-yaml-and-user-environment-layer.es.md), lo que dejaba al harness resolviendo valores orientados al usuario desde un `process.env` aplanado que ya no podía decir de dónde venía un valor. Se seguían tres consecuencias.

Una clave guardada a través de la página web seguía eclipsada por una clave más antigua en el `.env` del propio usuario, porque el provider de credenciales comparaba «el entorno» con su archivo y el entorno ahora incluía ese archivo. El callejón sin salida de migración que la separación debía eliminar simplemente se había movido.

Un endpoint podía ser redirigido por el proyecto. El `.env` del directorio de invocación se materializa como cualquier otra capa, y una URL base decide a dónde se envía una clave de API resuelta — así que un `DEEPSEEK_BASE_URL` escrito en un workspace que el modelo puede editar enviaría la credencial propia del usuario, y los prompts que llevan su código, a cualquier host que nombrara ese archivo. Nada en la vista aplanada podía distinguirlo de que el operador exportara la misma variable.

Y `!!js process.env.X` en la composición distribuida hacía que el mismo valor fuera alcanzable dos veces: una a través de la config de entrada y otra a través de la escalera que su consumer aplicara, con el ganador decidido por el orden de capas en lugar de por lo que significa el valor.

## Decision

**Un solo orden para los valores no secretos.** Todo valor configurable que no sea en sí mismo una credencial se resuelve en el mismo orden; los dominios difieren solo en qué niveles existen.

```text
explicit for this run     per-operation override, CLI argument
> user settings           settings.yaml
> composition             profile bundles, user patch layers, --patch overlays
> this launch's shell     inherited process environment
> discovered file         <invocation cwd>/.env, then $DSH_HOME/.env
> defaults                schema default, provider public default
```

Los ajustes están por encima de la composición porque eso es lo que hace el [seam de ajustes](2026-07-28-user-settings-seam.es.md): un plugin registra su config de entrada cordis como la capa `base` y las secciones del usuario se superponen sobre ella, y el seam no puede distinguir un valor que fijaron los bundles de un perfil de uno que fijó su capa de parche de usuario o un overlay de `--patch` — todos llegan como config de entrada. El CLI del producto no tiene palanca por encima de los ajustes almacenados, así que un despliegue que deba fijar un campo contra los ajustes de un usuario distribuye su propio bin o árbol de loader, o no monta ningún provider de ajustes. La composición sigue estando por encima del entorno, así que un `DEEPSEEK_BASE_URL` obsoleto en un shell no puede reescribir un endpoint configurado.

**Las credenciales conservan un orden más estrecho y separado**, y esta nota no las unifica:

```text
inherited process environment      (read-only, wins)
> $DSH_HOME/.credentials.yaml      (provider-managed, writable)
> <invocation cwd>/.env
> $DSH_HOME/.env
```

El entorno de lanzamiento gana porque `DEEPSEEK_API_KEY=… dsh`, un secreto de CI y un `-e` de contenedor son el único override que un operador debe poder aplicar por ejecución sin editar el estado de la máquina, y porque al no poder editarse desde dentro debe ser *visiblemente* de solo lectura. La configuración debe llevar solo la *referencia* — qué nombre resolver — y ese nombre sigue el orden de no secretos de arriba.

**El proyecto en el que se lanza el harness es de confianza, por defecto y sin prompt.** Un checkout puede llevar su propio endpoint, sus propias variables ordinarias y su propia clave; la clave está por debajo del almacén gestionado, así que una clave guardada a través de la página de Models nunca es desplazada por una que un checkout contenga por casualidad. `LaunchEnvironmentSnapshot.getFrom(name, sources)` sigue buscando solo en las capas que un llamador nombra, y omitir una es una negativa más que una degradación — el mecanismo existe para las decisiones donde una capa debe ser inalcanzable, no porque el proyecto sea una de ellas hoy.

**La confianza no se extiende a cambiar el propio harness.** `loadLayeredEnv` rechaza, al cargar y antes de materializar nada, cualquier `.env` que fije una variable que gobierne cómo arranca un proceso (`PATH`, `SHELL`, `NODE_OPTIONS`, `LD_PRELOAD`), qué programa ambiente maneja una operación (`EDITOR`, `PAGER`, `BROWSER`), qué código ejecuta un runtime antes del programa que se le pidió ejecutar (`BASH_ENV`, `PERL5OPT`, `PYTHONSTARTUP`, `RUBYOPT`, `JAVA_TOOL_OPTIONS`, los comandos de hook de Git), de dónde se cargan las instrucciones visibles para el modelo (todo el namespace `DSH_*`, `HOME`, `XDG_*`), o cómo se alcanza y se confía la red (variables de proxy y de CA). La comparación no distingue mayúsculas, así que `https_proxy` no es un rodeo.

La línea es que estas surten efecto sin ninguna acción del usuario, antes de cualquier turno, fuera de la política de permisos y del sandbox. `DSH_PERMISSION_MODE` apagaría las aprobaciones que hacen que confiar en un proyecto tenga sentido, y `BASH_ENV` ejecuta un archivo elegido por el proyecto en cada `bash -c` que emite la herramienta bash — el código del proyecto corriendo bajo la política del agent es el trato; el proyecto reescribiendo esa política no lo es. Enumerarlas es un juego perdido variable a variable, por eso se deniega todo el namespace `DSH_*` en lugar de un subconjunto auditado, y por eso la lista se organiza por lo que una variable *hace* en lugar de por qué runtime la posee. No hay opt-out: una puerta de escape tendría que poder leerse desde algún sitio, y cualquier cosa que un archivo descubierto pudiera fijar es el agujero mismo.

**`packages/util/launch-environment` posee la instantánea**, deliberadamente como utilidad en lugar de un seam de capacidad de tres paquetes. La instantánea se congela antes de que Cordis arranque y el lanzador la inyecta una sola vez, así que no hay implementación de runtime que intercambiar; los consumers necesitan tipos y funciones puras, que un paquete `util/` les da sin depender de un paquete de UI. `launchEnvironmentOf(ctx)` devuelve la instantánea del lanzador, o el entorno heredado como única capa — un host de SDK o un `cordis.yml` desnudo no descubrieron archivos, así que su capa única es de verdad con lo que se lanzó, y las mismas búsquedas de confianza siguen funcionando allí sin cambios.

**`verify-config-source-ownership`** es un alambre trampa estrecho para la forma ordinaria de una sola línea de un `apiKey`/`baseURL`/`headers` de entorno en línea en la configuración Cordis distribuida. Quitar esos inlines es lo que hace significativo el nivel de despliegue — con el árbol distribuido en silencio sobre `baseURL`, un valor presente significa que un humano o un despliegue lo fijó. Los adaptadores son dueños de la resolución real; la puerta no hace ninguna afirmación a nivel de repo sobre el acceso a `process.env`.

## Consequences

- El formulario web de credenciales ahora surte efecto frente a una clave más antigua en el `.env` del usuario; solo una clave exportada en el shell de lanzamiento la mantiene de solo lectura, y el diagnóstico lo dice.
- Un `.env` con `DSH_*`, `PATH`, `BROWSER` o una variable de proxy hace fallar el lanzamiento en lugar de aplicarse. Los desarrolladores que guardan switches en un `.env` de repositorio los mueven a su shell — una ruptura deliberada y sonora.
- La composición ya no es overridable por un endpoint de shell obsoleto. Sigue siendo overridable por el `settings.yaml` almacenado de un usuario, que es el layering del seam de ajustes y no algo que esta nota cambie; el CLI del producto no ofrece bandera por encima de él, así que un despliegue que deba ganar contra los ajustes almacenados posee su propio bin o árbol de loader.
- No resuelto: las capas siguen materializándose en `process.env`, así que las variables ordinarias del proyecto siguen llegando a los procesos hijos bajo el scrub de subprocesos. Las variables de bootstrap no pueden venir de un archivo en absoluto; el paquete de entorno registra el alcance restante de subprocesos como una limitación.
- Exa y Perplexity siguen capturando su clave en tiempo de carga en lugar de a través del seam de credenciales. Ya no leen `process.env` crudo — resuelven a través de las capas de confianza — pero convertirlos a resolución de credencial por petición es trabajo aparte.

## Alternatives considered

**Unificar las credenciales en el orden de no secretos, por quién redactó cada fuente.** Intentado y abandonado: se lee bien, pero el seam de ajustes ya fija la composición *por debajo* de la sección del usuario, así que «redactado por el despliegue» no es un nivel que el seam pueda expresar — y mover `.credentials.yaml` por encima del entorno de lanzamiento quitaría el único override del que dependen CI, contenedores y un `DEEPSEEK_API_KEY=…` por ejecución. Dos órdenes que explican cada uno su precedencia ganan a uno que no describe con precisión ninguno de los dos.

**Retener el enrutamiento y las credenciales del proyecto que invoca hasta que se confíe en él explícitamente.** Rechazado como postura del producto: un checkout es de confianza por defecto, sin prompt ni registro de confianza almacenado. El residuo es real y merece nombrarse — clonar un repositorio que lleva un `.env` que nombra otro endpoint o clave enruta esa sesión a través de él — y una puerta de confianza de proyecto posterior es donde se abordará, no una regla que haga que el caso común requiera ceremonia.

**Auditar una allowlist de variables `DSH_*` que un `.env` puede fijar.** Rechazado: la lista habría que reauditarla en cada switch nuevo, y el modo de fallo de olvidarse es silencioso. Denegar el namespace falla a salvo.

**Clasificar una variable de bootstrap por debajo de la capa de proceso en lugar de rechazarla.** Rechazado: `PATH` y `NODE_OPTIONS` no tienen un comportamiento de «perdedor» con sentido — un usuario que puso una en un `.env` cree que se aplica, e ignorarla en silencio es el fallo de «mi ajuste no tiene efecto» que esta decisión existe para eliminar.

**Construir la instantánea como un seam de capacidad de tres paquetes (`environment` / `environment-local` / consumers).** Rechazado por prematuro: el productor corre antes de que exista Cordis y no hay segunda implementación que seleccionar. La regla del repositorio es no dividir preventivamente.

**Dejar de materializar las capas en `process.env`.** Aplazado, no rechazado: mantendría las variables del proyecto fuera de los procesos hijos por completo, pero rompería en silencio cualquier capa de parche de usuario que lea `!!js process.env.X`. La instantánea ya es la autoridad de todo lo que resuelve el harness, así que esto puede aterrizar después sin cambiar ninguna escalera.
