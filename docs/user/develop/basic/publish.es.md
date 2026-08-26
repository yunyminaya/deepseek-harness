# Empaqueta e instala un plugin

[English](publish.md) | Español

Los tutoriales anteriores cargaron un plugin local a través de un overlay `--patch`. Este tutorial lo empaqueta como un **bundle** instalable, lo instala en un **profile** con `dsh plugin add` y explica el orden de las capas que determina la configuración compuesta. Asume que el CLI `dsh` está instalado. Completa primero la [configuración de plugin](./config.es.md).

Para usar en su lugar un checkout nuevo del código fuente, completa la [sección de ejecución desde el código fuente](../../../../README.es.md#run-from-source), mantén el directorio `hello-plugin` de este tutorial en la raíz del repositorio y ejecuta desde allí los comandos `dsh ...` restantes como `pnpm dsh ...`. Consulta la [ejecución desde el código fuente](../../../../apps/cli/reference/README.es.md#source-execution) para conocer el comportamiento de compilación y del launcher.

## Dos conceptos, dos manifests

La instalación se apoya en dos conceptos. Ambos se describen con un `package.json`, pero llevan tipos de manifest (manifiesto) distintos bajo la clave `dsh` y responden a preguntas diferentes:

- Un **bundle** es un paquete npm que incluye una capa de configuración. Su manifest declara `dsh.bundle` y responde a «¿qué aporta este paquete?»: un archivo patch que inserta o sobrescribe filas de plugin.
- Un **profile** es un directorio bajo `$DSH_HOME/profiles/<name>` que describe una composición ejecutable. Su manifest declara `dsh.profile` y responde a «¿qué bundles componen esta configuración y en qué orden?».

Un bundle es lo que tú creas y distribuyes; un profile es lo que un usuario inicia con `dsh --profile <name>`. Nada es ambas cosas.

### El manifest del bundle

Crea el directorio del paquete:

```sh
mkdir -p hello-plugin
```

```
hello-plugin/
├── package.json       # declares dsh.bundle
├── cordis.patch.yml   # the layer applied when a profile lists this bundle
└── index.js           # plugin modules the patch rows reference
```

Crea `hello-plugin/package.json`:

```json
{
  "name": "dsh-hello-plugin",
  "version": "0.1.0",
  "type": "module",
  "main": "index.js",
  "files": ["index.js", "cordis.patch.yml"],
  "dsh": { "bundle": { "patch": "./cordis.patch.yml" } }
}
```

Crea `hello-plugin/index.js` con el punto de entrada del plugin:

```js
export const name = 'hello-plugin'

export function apply() {
  console.log('[hello-plugin] plugin loaded!')
}
```

Crea `hello-plugin/cordis.patch.yml`. El patch es un array de YAML como los overlays `--patch` que has estado escribiendo, salvo que las filas de plugin referencian el paquete por nombre en lugar de por una ruta relativa de código fuente, para que la resolución de Node encuentre el código instalado:

```yaml
- insert:
    - id: hello
      name: dsh-hello-plugin
```

Un paquete sin la declaración `dsh.bundle` también se instala, pero solo como dependencia plana: `dsh plugin` imprime una advertencia y no activa ninguna capa. Usa ese formato de paquete para una librería que los paquetes de plugin importen, no para un plugin que los usuarios habiliten.

### El manifest del profile

Un directorio de profile contiene dos archivos:

- `package.json` — las dependencias de plugin fuera del árbol del profile (gestionadas por pnpm) más el manifest `dsh.profile` con su lista ordenada de `bundles`.
- `cordis.patch.yml` — la capa de patch propia del usuario, aplicada después de cada capa de bundle.

Nunca escribes un manifest de profile a mano: `dsh plugin` lo crea y lo mantiene. La siguiente sección muestra el resultado.

## Instala en un profile

`dsh plugin --profile <name> <args...>` reenvía a pnpm en el directorio del profile, así que todos los verbos de pnpm funcionan. Desde el directorio que contiene `hello-plugin`, instala el checkout del paquete:

```sh
dsh plugin --profile demo add ./hello-plugin
```

El primer uso inicializa el profile (con `@deepseek-ai/dsh-base` como su primer bundle), pnpm enlaza el checkout y `dsh` añade el bundle a `dsh.profile.bundles` porque el paquete declara `dsh.bundle`:

```json
{
  "name": "dsh-profile-demo",
  "private": true,
  "dependencies": {
    "dsh-hello-plugin": "link:/path/to/hello-plugin"
  },
  "dsh": {
    "profile": {
      "bundles": [
        "@deepseek-ai/dsh-base",
        "dsh-hello-plugin"
      ]
    }
  }
}
```

Verifica la capa sin arrancar y luego arranca:

```sh
dsh --profile demo --dump-config   # shows a "# == dsh-hello-plugin" layer
dsh --profile demo
```

`dsh plugin --profile demo remove dsh-hello-plugin` elimina tanto la dependencia como la capa.

## El orden de carga

La configuración efectiva se compone sobre una raíz vacía aplicando, en orden:

1. Cada patch de bundle nombrado en la lista `dsh.profile.bundles` del profile, en el orden de la lista — `@deepseek-ai/dsh-base` primero y después cada bundle instalado en el orden en que se añadió.
2. El `cordis.patch.yml` propio del profile.
3. El `$DSH_HOME/cordis.patch.yml` del nivel de home — preferencias locales de la máquina compartidas por todos los profiles.
4. Cada overlay `--patch <path>`, en el orden de argv.

Los argumentos de la app no son otra capa de patch. Un surface bundle puede resolverlos a través de un servicio ordinario propiedad de la app, descrito más abajo.

Las capas posteriores ganan por fila, y un patch reemplaza el valor completo de `config` de una fila en lugar de fusionar las claves en profundidad. Dos consecuencias para los autores de bundles:

- Tu patch puede sobrescribir filas de capas anteriores por `id` — igual que [el bundle `dsh-web-app`](../../../../packages/bundle/web-app/cordis.patch.yml) sobrescribe filas de `dsh-base` — pero debe reafirmar todas las claves que la fila necesita, no solo la modificada.
- Los usuarios pueden sobrescribir tus filas en el `cordis.patch.yml` de su profile sin tocar tu paquete; por eso, prefiere valores por defecto que los usuarios probablemente conserven y deja que el schema cargue con el resto.

Los nombres de bundle incluidos se resuelven siempre desde la propia instalación de dsh; pnpm gestiona solo los paquetes fuera del árbol, así que tu bundle puede confiar en que `@deepseek-ai/dsh-base` esté presente y actualizado.

## Dale a un surface bundle su propia línea de comandos

Un bundle que define una app ejecutable monta un plugin de provider ordinario:

```yaml
- id: hello-startup
  name: 'dsh-hello-plugin/startup'
```

El plugin exporta `inject = ['cmdlineArgs']`, llama a `parseCmdline` de [`@deepseek-ai/dsh-cmdline`](../../../../packages/boot/cmdline/README.es.md) con su propio programa commander y proporciona su servicio propiedad de la app desde la acción del programa. El launcher entrega a cada plugin los mismos argumentos inmutables después de las banderas del launcher, así que las banderas específicas de la app no requieren cambios en el launcher y varios plugins pueden analizar la instantánea. La fila del Loader no necesita marcador de launcher ni un tipo especial.

Las filas configuradas por esos argumentos inyectan el servicio del provider y lo leen de sus propias opciones `!!js`, con el valor de despliegue al lado como respaldo:

```yaml
- id: my-app
  name: '@example/my-app'
  inject: [myAppStartup]
  config:
    port: !!js ctx.myAppStartup.port ?? 8080
```

Con `--help`, el provider no publica ningún servicio, así que esas filas nunca se activan. El Loader monta la composición una vez, espera las inyecciones ordinarias de cada fila y solo entonces evalúa la configuración `!!js` de esa fila contra su contexto inyectado.

## Instalar desde GitHub: la trampa del script de build

Publicar en un registro no es obligatorio — los usuarios pueden instalar directamente desde un host de git:

```sh
dsh plugin --profile demo add github:you/hello-plugin
```

Pero una instalación desde git obtiene **fuentes, no artefactos compilados**: nada ejecuta tu script `build`, así que un paquete de TypeScript llega sin su salida `lib/` y falla al cargar. Deben ocurrir dos cosas, una en cada lado:

- **El autor** incluye un script `prepare` — pnpm lo ejecuta después de una instalación desde git — que compila los puntos de entrada publicados desde el código fuente, de forma autocontenida: no debe asumir un contexto solo de desarrollo, como un checkout hermano del monorepo. [turtle-ui](https://github.com/deepseek-harness/turtle-ui) es un ejemplo funcional: su `prepare` ejecuta una configuración de tsdown dedicada que transpila `src/` sin referencias de proyecto ni comprobación de tipos.
- **El usuario** añade el build a la lista blanca. pnpm ≥10 se niega a ejecutar el script `prepare` de una dependencia de git hasta que se permita explícitamente, así que el primer `add` falla; `dsh` señala la solución — copia la clave exacta del paquete que pnpm imprimió en el `pnpm-workspace.yaml` del profile:

  ```yaml
  allowBuilds:
    dsh-hello-plugin: true
  ```

  y vuelve a ejecutar el `add`.

Trata ese permiso como lo que es: **permiso para ejecutar el código del paquete en tu máquina en el momento de la instalación**, fuera de cualquier sandbox bajo el que se ejecute el agent. Solo permite paquetes cuya fuente confíes, y fija un commit (`github:you/hello-plugin#<sha>`) para que un push posterior no pueda cambiar silenciosamente lo que se ejecuta.

Si prefieres no pedir a los usuarios ese permiso, distribuye en su lugar artefactos compilados — ninguna de las dos formas necesita permiso de build:

- **Publica en npm** con `lib/` compilada en el momento de `pnpm publish`; `dsh plugin add your-package` instala entonces el código precompilado.
- **Distribuye un tarball** desde `pnpm pack`; los usuarios ejecutan `dsh plugin add ./hello-plugin-0.1.0.tgz`.

## Pasos siguientes

- [Plugins y ciclo de vida](../framework/index.es.md) — el ciclo de vida completo del plugin
- [Referencia de comportamiento del CLI](../../../../apps/cli/reference/README.es.md) — precedencia exacta de capas, banderas y mecánica de profiles
