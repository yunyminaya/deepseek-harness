# Recetario: añadir una tarjeta de ajustes

[English](adding-a-settings-card.md) | Español

Cómo un plugin pone su propia configuración en la página de ajustes web. Nada en esta ruta necesita un cambio dentro de este repositorio: el Host sirve cada namespace de ajustes registrado, y la sección **Plugins** indexa sus tarjetas por el namespace que editan, así que un plugin que registra ambas mitades queda emparejado automáticamente.

Las dos mitades viven en un único paquete — la mitad del Host bajo `src/`, la mitad del navegador bajo `src/client/`, exportada como `./client` y declarada con `dsh.client`. [`packages/client/ui-theme`](../../packages/client/ui-theme) es un ejemplo resuelto de ese empaquetado; las tarjetas que publica esta sección viven en [`packages/client/ui-settings-plugins`](../../packages/client/ui-settings-plugins).

## 1. Registra el namespace (mitad Host)

El namespace es la clave de unión, así que elígela una vez y escríbela igual en ambas mitades. Un consumidor que ya tiene una entrada `cordis.yml` debería registrarse a través de `installSettingsSection`, que superpone la entrada bajo el documento del usuario y sigue funcionando cuando no hay ningún provider de ajustes montado:

```ts
import type { Context } from '@deepseek-ai/cordis'
import { installSettingsSection, settingsNamespace } from '@deepseek-ai/dsh-settings'
import z from '@deepseek-ai/schemastery'

declare function assertReachable(endpoint: string | undefined): void
declare function rebuildFromSettings(config: Config): void

export const MY_PLUGIN_NS = settingsNamespace('my-plugin')

export interface Config {
  endpoint?: string
  retries?: number
}

export const Config: z<Config> = z.object({
  endpoint: z.string(),
  retries: z.number().step(1).min(0).default(3),
})

export function apply(ctx: Context, config: Config) {
  let source = () => config
  installSettingsSection(ctx, MY_PLUGIN_NS, Config, config, {
    // Constraints the schema cannot express refuse the write, not the next use.
    validate: value => void assertReachable(value.endpoint),
    setSource: (current) => { source = current },
    onChange: () => { rebuildFromSettings(source()) },
  })
}
```

`role('secret')` en un campo mantiene su valor fuera de cada respuesta; la tarjeta escribe ese campo en un payload `update`/`mutate`, o aborda una referencia de credenciales a través del dominio `credentials`. `applies: 'restart'` indica a una superficie de configuración que el propietario actúa sobre un cambio solo en el siguiente arranque.

## 2. Registra la tarjeta (mitad navegador)

La tarjeta se registra en `settings.plugin.item` bajo su namespace y posee todo lo que hay dentro — chrome, controles y texto. Lee y escribe a través de `ctx.settingsScope`, que vincula cada escritura a la revisión que leyó:

```ts ignore-check
import type { ClientContext } from '@deepseek-ai/dsh-client-runtime/client'
// Type-only: the keyed slot's declaration. Cross-plugin collaboration goes
// through cordis services; a value import fails the client bundle-purity gate.
import type {} from '@deepseek-ai/dsh-client-ui-settings-plugins/client'

export const inject = ['slots', 'locale', 'connection', 'remote', 'settingsScope']

export function apply(ctx: ClientContext): void {
  const card = new MyPluginCardController(ctx.settingsScope.bind({ namespace: 'my-plugin' }))
  ctx.slots.inject('settings.plugin.item', () => ctx.slots.register({
    name: 'settings.plugin.item',
    key: 'my-plugin',
    locale: 'settings.myPlugin',
    inject: () => card.inject(),
  }, MyPluginCard),
  )
}
```

La instantánea del scope lleva lo que un formulario necesita: el `value` resuelto, la `base` de composición y la capa `user` cruda, cuya **presencia** de clave — no su valor — es lo que marca un campo como sobrescrito. `scope.set(field, value)` almacena un campo y `scope.unset(field)` lo restablece a la capa de composición.

## 3. Qué hace la pestaña con ello

La pestaña **Plugin configuration** lee qué namespaces sirve el Host y despacha una clave de slot por namespace. Una tarjeta se renderiza cuando el Host sirve su clave y se omite cuando no la sirve, así que un despliegue que nunca compuso la mitad del Host no muestra rastro de la tarjeta. Un namespace servido que ninguna tarjeta reclama no renderiza nada — así es como los namespaces de otras páginas (`ui-theme`, `permission`, `llm-*`) se mantienen fuera de esta pestaña.

Las tarjetas aparecen en el orden en que se registraron en el slot; una entrada con clave no declara `order` propio.

## Empaquetado

La mitad del navegador se sirve a la página mediante el [sistema de módulos del cliente](../../packages/client/modules), que escanea las entradas de Loader habilitadas en busca de paquetes que declaren `dsh.client` y sirve la exportación `./client` compilada de cada uno. Así, el plugin aparece en la página en cuanto un `cordis.yml` lo monta — sin recompilar la aplicación web.

```jsonc
{
  "exports": {
    ".": { "types": "./lib/types/index.d.ts", "default": "./lib/index.js" },
    "./client": { "types": "./lib/types/client/index.d.ts", "default": "./lib/client.js" }
  },
  "dsh": { "client": { "platform": "web", "inject": ["@deepseek-ai/dsh-client-ui-settings-plugins"] } }
}
```

El bundle debe ser el artefacto de fábrica lazy-CJS del loader. Dentro de este repositorio, `tsdown.config.ts` son tres líneas sobre el preset compartido:

```ts ignore-check
import { clientBundle } from '../tsdown.client.ts'

export default clientBundle('@deepseek-ai/dsh-client-my-plugin', ['lib/types/index.js', 'lib/types/invariant.js'])
```

Ese preset no se publica hoy, así que un paquete fuera de este repositorio tiene que reproducir por sí mismo el mismo formato de salida. El gate de pureza del bundle también rechaza los imports de valor entre plugins, así que una tarjeta no puede importar el chrome de tarjeta de esta sección ni su modelo de formulario en etapas — renderiza el suyo propio y posee su propio staging y su propia comprobación de revisiones. Ambos límites están registrados en [las limitaciones conocidas de la sección](../../packages/client/ui-settings-plugins/README.es.md#known-limitations-and-deferred-work).
