# Sistema de eventos

[English](events.md) | Español

Los eventos son el mecanismo de comunicación central entre los plugins de Cordis. Harness los usa de forma extensiva para puntos de extensión débilmente acoplados.

## Uso básico

### Escucha un evento

```ts ignore-check
ctx.on('event-name', (payload) => {
  // Handle the event.
})
```

### Emite un evento

```ts ignore-check
ctx.emit('event-name', payload)
```

## Modos de evento

Cordis ofrece varios modos de evento para distintos contratos de interacción.

### emit — difusión

Cada listener se ejecuta de forma síncrona y los valores de retorno se ignoran:

```ts ignore-check
// Emit
ctx.emit('my-plugin/ready', { id: 'worker-1' })

// Listen
ctx.on('my-plugin/ready', ({ id }) => {
  console.log(`${id} is ready`)
})
```

### bail — cortocircuito

Los listeners se ejecutan en orden; el primer resultado distinto de `null`, `false` o `undefined` se convierte en el resultado final:

```ts ignore-check
// Dispatch
const result = ctx.bail('some-check', input)

// Listen: a returned value stops later listeners.
ctx.on('some-check', (input) => {
  if (shouldBlock(input)) return 'blocked'
  // Return null, false, or undefined to continue to the next listener.
})
```

### serial — ejecución ordenada

Los listeners se ejecutan en orden de registro y los resultados asíncronos se esperan. El primer resultado distinto de `null`, `false` o `undefined` detiene la ejecución posterior:

```ts ignore-check
await ctx.serial('setup-phase', context)
```

### waterfall — procesamiento en cadena

En el modo waterfall (cascada de eventos), cada listener puede envolver el resultado descendente para formar una cadena de procesamiento. Un listener **debe llamar a `next()` para delegar hacia abajo**; omitir la llamada cortocircuita el pipeline:

```ts ignore-check
// Dispatch
const output = await ctx.waterfall('my-plugin/transform', input, async () => input)

// Listen: next() is mandatory.
ctx.on('my-plugin/transform', async (_input, next) => {
  const downstream = await next()
  return downstream.trim()
})
```

::: warning
Un listener de waterfall **debe llamar a `next()`**. Omitirla cortocircuita el pipeline por diseño, lo que permite la intercepción y el comportamiento de gateway.
:::

## Eventos tipados

Harness usa la fusión de declaraciones de TypeScript para eventos seguros por tipos:

```ts
import '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Events {
    'my-plugin/ready': (payload: { id: string }) => void
    'my-plugin/check': (input: string) => boolean | undefined
    'my-plugin/transform': (input: string, next: () => Promise<string>) => Promise<string>
  }
}

// ctx.on('my-plugin/ready', ...) and ctx.emit('my-plugin/ready', ...)
// are now inferred correctly.
```

## Eventos de Cordis y registros de sesión

Los eventos de Cordis de Harness usan nombres `namespace/action`, incluidos `agent/step`, `agent/request`, `agent/request-error`, `tools/result` y `session/event`. Las regiones generadas `cordis-surface` en las [páginas de subsistemas](../../../subsystems/core.es.md) registran las firmas y los modos completos.

`turn/*`, `step/*`, `tool/call`, `tool/result` y `compaction/*` son tipos de eventos de sesión duraderos, no eventos de Cordis con el mismo nombre. Para observarlos, escucha `session/event` e inspecciona `event.type`.

## Los listeners de eventos son effects

Un listener registrado con `ctx.on()` se elimina automáticamente cuando su plugin se descarga:

```ts ignore-check
export function apply(ctx: Context) {
  // This listener is removed when the plugin disposes.
  ctx.on('tools/result', handler)
}
```

## Ejemplo: plugin de registro

Este plugin registra las llamadas de herramienta y sus resultados:

```ts
import type { Context } from '@deepseek-ai/cordis'
import '@deepseek-ai/dsh-tools'

export const name = 'tool-logger'

export function apply(ctx: Context) {
  ctx.on('tools/result', (exec, result) => {
    console.log(`[tool] ${exec.name}(${JSON.stringify(exec.arguments)})`)
    const text = result.content
      .map(block => block.type === 'text' ? block.text : '')
      .join('')
    console.log(`[tool result] ${text.slice(0, 100)}`)
  })
}
```

## Pasos siguientes

- [Capas de capacidad](../practice/index.es.md) — comprende los eventos dentro de las interfaces de capacidad
- [Adaptadores de LLM](../practice/llm-adapter.es.md) — implementa un backend de LLM completo
