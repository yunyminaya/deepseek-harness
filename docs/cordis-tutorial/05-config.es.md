# 5. Configuración

[English](05-config.md) | Español

Cada entrada de `cordis.yml` puede llevar un bloque `config`, y el plugin declara un schema que la valida antes de que se ejecute `apply`. Una config mala hace fallar la carga con un error preciso: el plugin nunca arranca a medio configurar.

## Un plugin configurable

Crea `config-demo.ts` en `tmp/cordis-tutorial`:

```ts
import type { Context } from '@deepseek-ai/cordis'
import Schema from '@deepseek-ai/schemastery'

export const name = 'config-demo'

export interface Config {
  greeting: string
  targets: string[]
}

export const Config: Schema<Config> = Schema.object({
  greeting: Schema.string().default('Hello'),
  targets: Schema.array(String).default(['world']),
})

export function apply(ctx: Context, config: Config) {
  for (const target of config.targets) {
    console.log(`${config.greeting}, ${target}!`)
  }
}
```

El `Config` exportado es a la vez una interfaz de TypeScript y un schema de runtime con el mismo nombre —los consumidores obtienen el tipo y Cordis, el validador. Este repositorio usa [Schemastery](https://github.com/shigma/schemastery) para los schemas; Cordis en sí acepta cualquier validador [Standard Schema](https://standardschema.dev/), así que un objeto plano exportado como `Config` no funcionará.

Configúralo:

```yaml
- name: './config-demo.ts'
  config:
    targets: ['alpha', 'beta']
```

Ejecuta:

```
Hello, alpha!
Hello, beta!
```

Se omitió `greeting`, así que el valor por defecto del schema lo rellenó: `apply` siempre recibe una config completa y validada.

## Falla ruidosamente

Ahora pásale algo inválido:

```yaml
- name: './config-demo.ts'
  config:
    targets: 'not-an-array'
```

```
ValidationError: invalid config:
  - $.targets expected array but got not-an-array (at targets)
```

La fiber del plugin pasa a FAILED, y el launcher de este tutorial termina con el estado 1 tras imprimir el error. Un plugin también debería rechazar una config válida según el schema que nombre un recurso o provider no disponible, en cuanto pueda resolver esa referencia.

## Valores de config calculados

El loader que usa este repositorio admite una etiqueta `!!js` para los valores de config que deben calcularse en el momento de la carga:

```yaml
- name: './config-demo.ts'
  config:
    greeting: !!js process.env.DEMO_GREETING ?? 'Hello'
```

`!!js` solo funciona dentro de `config` y en el campo `disabled` de una entrada. `disabled: !!js ...` se evalúa contra el contexto del loader en cada decisión de montaje (una extensión de este repositorio), así que una fila puede condicionarse por plataforma o entorno; el resto de metadatos (`name`, `id`, `inject`, ...) permanece estático, donde una expresión es un dato normal con valor de verdad. Consulta [configuración del loader](../cordis-primer.es.md#loader-configuration).

Siguiente: [Composición y HMR](06-composition-and-hmr.es.md) — tratar `cordis.yml` como la aplicación.

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)
