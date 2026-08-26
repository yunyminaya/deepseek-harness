# Configuración de plugin

[English](config.md) | Español

Acepta la configuración suministrada a través de `cordis.yml`.

## Define el tipo Config

Exporta un tipo `Config` y un schema de Schemastery con el mismo nombre. Pon los valores por defecto directamente en los campos del schema:

```ts
import type { Context } from '@deepseek-ai/cordis'
import Schema from '@deepseek-ai/schemastery'

export const name = 'my-plugin'

export interface Config {
  greeting: string
  maxRetries: number
  verbose?: boolean
}

export const Config: Schema<Config> = Schema.object({
  greeting: Schema.string().default('Hello'),
  maxRetries: Schema.number().default(3),
  verbose: Schema.boolean().default(false),
})

export function apply(ctx: Context, config: Config) {
  console.log(config.greeting)  // User value or schema default.
}
```

Añade la configuración a la fila del plugin local insertado en `scratch-plugin/cordis.yml`:

```yaml
- insert:
    - id: hello
      name: './src/my-plugin.ts'
      config:
        greeting: 'Hi there'
        maxRetries: 5
```

Al cargar el plugin, Cordis usa el schema exportado para validar la configuración y rellenar los valores por defecto. No exportes un objeto plano como `Config`; no implementa la interfaz Standard Schema que Cordis requiere.

## Validación del schema

Usa Schemastery para expresar una validación más estricta:

```ts
import type { Context } from '@deepseek-ai/cordis'
import Schema from '@deepseek-ai/schemastery'

export const name = 'validated-plugin'

export interface Config {
  apiKey: string
  timeout: number
  mode: 'fast' | 'accurate'
}

export const Config = Schema.object({
  apiKey: Schema.string().required(),
  timeout: Schema.number().default(30000),
  mode: Schema.union(['fast', 'accurate']).default('fast'),
})

export function apply(ctx: Context, config: Config) {
  // config is validated and type-safe.
}
```

El schema se ejecuta mientras el plugin se carga. Una configuración inválida hace fallar la carga con un error accionable.

## Principios de diseño

### No fijes en código los valores ajustables

Harness exige que **todo lo que dos despliegues puedan querer configurar de forma distinta sea un campo de configuración**.

```ts
// Wrong: hardcoded timeout.
const TIMEOUT = 30000

// Correct: configurable.
export interface Config {
  timeoutMs: number  // Defaults to 30000.
}
```

La prueba es si `cordis.yml` puede cambiar el valor sin editar código.

### Falla de forma ruidosa con una configuración inválida

Expresa las restricciones autocontenidas en el schema para que una configuración inválida falle mientras el plugin se carga. Las referencias a servicios o recursos registrados requieren inyección de dependencias; el [tutorial de servicios](../framework/service.es.md) presenta ese contrato.

## Trabaja con HMR

Una edición de configuración reemplaza el plugin en caliente: el framework descarga la instancia antigua y carga una nueva. Como los registros son effects y se limpian solos, el reemplazo no conserva los registros de la instancia antigua.

## Pasos siguientes

- [Empaqueta e instala un plugin](./publish.es.md) — distribuye el plugin como paquete instalable
- [Plugins y ciclo de vida](../framework/index.es.md) — comprende el ciclo de vida completo del plugin
- [Servicios y dependencias](../framework/service.es.md) — proporciona un servicio a otros plugins
