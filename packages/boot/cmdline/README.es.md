# `@deepseek-ai/dsh-cmdline`
[English](README.md) | Español

La línea de comandos que un lanzador de dsh entrega a la app que arranca. El lanzador analiza solo sus propios flags (`--profile`, `--patch`, los dumps de configuración) y entrega **todo lo que viene después** al árbol tal cual, de modo que una app es dueña de su familia de flags, de su texto `--help` y de sus errores de análisis en lugar de que el lanzador los conozca.

## Los valores del lanzador

Un lanzador llama a `provideCmdline(ctx, host)` antes de que monte cualquier entrada del árbol, lo que aporta:

- `ctx.cmdlineArgs` — los argumentos internos de la invocación. `get()` es toda la interfaz, y devuelve una instantánea: `dsh --profile tui --resume abc` produce `['--resume', 'abc']`.
- `ctx.appExit` — una solicitud de salida de proceso acotada, conectada al controlador de apagado del lanzador.

Un host embebido sin línea de comandos aporta una lista vacía; esa es la respuesta honesta, no un valor ausente.

## Providers ordinarios y configuración inyectada

Cualquier plugin de la app puede inyectar `cmdlineArgs`, analizarlo y publicar un servicio ordinario propiedad de la app. `parseCmdline(ctx, program)` es solo un adaptador de commander; la acción del propio programa es dueña de la validación y del servicio publicado:

```ts ignore
export const name = 'web-startup'
export const inject = ['cmdlineArgs']

export function apply(ctx: Context): void {
  const program = webCommand()
  program.action(() => ctx.provide('webStartup', webValuesFrom(program)))
  parseCmdline(ctx, program)
}
```

Su fila de Loader no lleva marcador de lanzador ni tipo especial:

```yaml
- id: web-startup
  name: '@deepseek-ai/dsh-web-app/startup'
```

Toda fila configurada a partir de esos valores usa inyección de servicios ordinaria y acceso perezoso directo a la configuración:

```yaml
- id: webserver
  name: '@deepseek-ai/dsh-host-webserver'
  inject: [webStartup]
  config:
    host: !!js ctx.webStartup.host ?? '127.0.0.1'
    port: !!js ctx.webStartup.port ?? 3080
```

`parseCmdline` se niega en la carga a un programa en el que ningún comando declara una acción, enruta la salida y el exit de cada comando a través del lanzador (commander copia esos ajustes en los subcomandos solo en el registro), y analiza los argumentos inmutables; commander ejecuta la acción síncrona del comando invocado en caso de éxito. Una acción rechaza una invocación inválida con `program.error(...)` — antes de publicar, ya que las sentencias anteriores al rechazo ya se han ejecutado. En `--help`, `--version`, un error de análisis o ese rechazo, el helper escribe el texto de commander y solicita la salida; el provider no publica nada, así que las filas dependientes nunca se activan.

### Cómo ordena la inyección la configuración

El Loader difiere la interpolación `!!js` de una fila hasta que las inyecciones declaradas de esa fila están activas, y luego evalúa contra el contexto de plugin de la fila. El ejemplo anterior puede por tanto leer `ctx.webStartup` directamente: Cordis ya ha poblado ese servicio inyectado antes de que el Loader pida la configuración de `webserver`. Los árboles de include preservan los nodos de expresión anidados hasta que cada fila objetivo llega a ese punto. El reemplazo de providers y la recarga de patches en vivo repiten la interpolación contra los servicios inyectados actuales, de modo que un flag de lanzamiento no puede reiniciarse en silencio.

### Argumentos inmutables compartidos

`get()` no consume ni muta argv. Varios plugins pueden analizar la misma instantánea y aportar servicios de forma independiente. El lanzador no inspecciona la composición en busca de un dueño de la línea de comandos; un profile sin lector simplemente ignora sus argumentos de app.

Un plugin fuera del árbol trae su propia copia de commander, de modo que los errores de flujo de control de commander se detectan estructuralmente y no por identidad de clase; una comprobación de identidad relanzaría un help impreso como fallo de carga fatal.

## Experiencia de modelo

Ninguna, ya que este paquete resuelve la línea de comandos del propio proceso antes de que exista sesión alguna.

#### Efecto de KV Cache

Ninguno; este paquete ni ensambla ni envía una petición al provider.

## Limitaciones conocidas y trabajo diferido

- **Los flags del lanzador deben preceder a los argumentos de la app.** La división es posicional: el primer token que el lanzador no reconoce inicia los argumentos internos, así que un `--patch` colocado después de un flag de la app pertenece a la app. El analizador del lanzador consume un único `--`, de modo que un argumento de app que deba sobrevivir como un `--` literal necesita `-- --`.
- **Un servicio propiedad de la app no tiene provider declarado estáticamente.** Las filas consumidoras lo nombran mediante inyección ordinaria; un bundle que omita su provider falla en el asentamiento con entradas pendientes que nombran el servicio, no en la carga.
- **Un patch de usuario que reemplaza todo el `config` de una fila elimina sus expresiones.** Un flag gana al valor escrito junto a él, no a un literal que el usuario escribió en lugar de la expresión; conservar la expresión es lo que mantiene ganando al flag.
