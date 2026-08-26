# Servicios y dependencias

[English](service.md) | Español

Un servicio es una capacidad que un plugin expone a otros plugins. `inject` declara los servicios que un plugin requiere.

## ¿Qué es un servicio?

En Harness, `tools`, `llm` y `agents` son servicios. Cada uno es una capacidad con nombre montada en `ctx`:

```ts ignore-check
ctx.tools    // ToolRuntime service
ctx.llm      // LLM service
ctx.agents   // Agent service
```

Cualquier plugin puede proporcionar un servicio para que otros plugins lo consuman.

## Consume un servicio

Declara `inject` para usar un servicio existente:

```ts ignore-check
export const inject = ['tools']

export function apply(ctx: Context) {
  // ctx.tools exists and is ready here.
  ctx.tools.register(/* ... */)
}
```

Cuando `apply` se ejecuta, todos los servicios declarados por `inject` están listos. Si un servicio no está listo, el plugin espera en lugar de ejecutarse.

## Proporciona un servicio

### Extiende Service

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

export default class MetricsService extends Service {
  static inject = ['llm']  // A service may depend on other services.

  constructor(ctx: Context) {
    super(ctx, 'metrics')  // 'metrics' is the service name.
  }

  // Public service method.
  record(event: string, value: number) {
    // ...
  }
}
```

Después de cargar este plugin, los consumers acceden al servicio como `ctx.metrics`:

```ts ignore-check
export const inject = ['metrics']

export function apply(ctx: Context) {
  ctx.metrics.record('tool_call', 1)
}
```

### Declara su tipo

Usa la fusión de declaraciones de TypeScript para tipar `ctx.metrics`:

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Context {
    metrics: MetricsService
  }
}

export default class MetricsService extends Service {
  constructor(ctx: Context) {
    super(ctx, 'metrics')
  }

  record(event: string, value: number) { /* ... */ }
}
```

## Comportamiento de las dependencias

### Dependencias requeridas y opcionales

```ts ignore-check
// Required: the plugin does not load while the service is absent.
export const inject = ['tools']

// Optional: omit inject and query with ctx.get() at the use site.
export function apply(ctx: Context) {
  const metrics = ctx.get('metrics')
  metrics?.record('plugin_loaded', 1)
}
```

### Cuando un servicio desaparece

Si un servicio requerido desaparece mientras la aplicación se ejecuta, por ejemplo porque su provider se descarga:

1. Los plugins dependientes se liberan automáticamente.
2. Se cargan de nuevo cuando el servicio regresa.

Esto evita que un plugin llame a un servicio que ya no existe.

## Aislamiento de servicios

`cordis.yml` puede aislar servicios para que grupos de plugins separados vean instancias separadas del mismo servicio:

```yaml
- id: group-a
  name: '@deepseek-ai/cordis-plugin-group'
  group: true
  isolate:
    shell: true
  config:
    - name: '@deepseek-ai/dsh-bash-local'
      config:
        timeoutMs: 5000
    - name: './src/plugin-a.ts'

- id: group-b
  name: '@deepseek-ai/cordis-plugin-group'
  group: true
  isolate:
    shell: true
  config:
    - name: '@deepseek-ai/dsh-bash-local'
      config:
        timeoutMs: 60000
    - name: './src/plugin-b.ts'
```

`plugin-a` y `plugin-b` ven cada uno la instancia de Bash de su propio grupo, sin efectos entre grupos.

## Servicios integrados de Harness

El repositorio genera los nombres de servicio, los métodos públicos y las ubicaciones del código fuente en la [página de subsistemas](../../../subsystems/core.es.md) de cada servicio. Usa esas regiones generadas y la interfaz de TypeScript del servicio mientras desarrollas un plugin; no mantengas una segunda lista estática.

## Pasos siguientes

- [Sistema de eventos](./events.es.md) — comunícate entre plugins sin acoplamiento fuerte
- [Capas de capacidad](../practice/index.es.md) — usa los servicios como interfaces de capacidad
