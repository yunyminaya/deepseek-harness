# Agent Note: Recortar los seams de línea de comandos hasta las interfaces existentes

Status: implemented

[English](2026-08-11-cmdline-seam-trim.md) | [中文](2026-08-11-cmdline-seam-trim.zh.md) | Español

## Problema

La línea de comandos propiedad de la app ([note](2026-08-06-app-owned-command-line.es.md)) se publicó con tres seams más anchos de lo que sus consumidores necesitaban: una máquina de estados de activación de filas en memoria vendida (`Entry.enableRuntime` más `enableRow` exportado desde `dsh-cmdline`, un paquete de línea de comandos dueño de un concepto de Loader) cuyo único propósito era la fila de recarga condicional de `--dev`; un símbolo de protocolo vendido `EntryConfigResolver` cuyo único implementador era Include; y un launcher que aún reconocía la fila `headless-runner` para elegir los códigos de salida SIGTERM, compuertar la vigilancia de parches de usuario y proporcionar un seam `headlessIo` que duplicaba `ctx.appExit`.

## Decisión

Expresar los tres con interfaces que ya existen:

- **Sin fila dev condicional.** La cadena de recarga deja de ser condicional: `dsh-web-app` monta la fila `client-hmr` incondicionalmente y se eliminan `--dev`, junto con el modo `mode` del runtime web, el contrato de prompt bifurcado por modo y la variable bash `DSH_WEB_MODE`. Sin un watcher de rebuild (`pnpm run dev:web`) reescribiendo los bundles del cliente, la cadena sondea archivos sin cambios y permanece inactiva, de modo que la fila siempre activa cuesta un intervalo de stat-poll y una ruta SSE. `Entry.enableRuntime`, sus dos campos de estado y `enableRow` se eliminan sin nada que los sustituya.
- **Config del carrier de árbol.** Include declara el marcador existente `EntryGroup.key` en lugar de implementar `EntryConfigResolver`; el hook de Loader conserva el literal de configuración de cada carrier de árbol. El propio `path` de Include pierde el soporte de `!!js` — ninguna configuración lo usó jamás, y la prueba de fijación ahora afirma el contrato literal de carrier de árbol en su lugar.
- **Conocimiento de app del launcher.** El launcher no reconoce ninguna fila de app. SIGTERM es una solicitud de detención ordinaria del supervisor y sale con 0 en todas las superficies (SIGINT sigue siendo 130); el launcher no puede saber si la app consideró su trabajo completo, y el 143 anterior dependía de nombrar la fila headless. Cada arranque vigila sus capas de parche de usuario — una superficie de un solo uso sale por un apagado acotado, que dispone los watchers antes de que el loop se drene. El runner headless sale por `ctx.appExit` como cualquier otra app; sus flujos de salida son un seam de prueba interno de paquete `internals`, y `ctx.headlessIo` se elimina.

## Alternativas consideradas

- **Conservar `enableRuntime` pero mover `enableRow` fuera de `dsh-cmdline`**: la reubicación arregla el límite del paquete pero conserva la máquina de estados vendida cuyas semánticas (sobrevive a la reaplicación, rollback ante fallo) deben re-derivarse en cada sync upstream.
- **`entry.update({ disabled: null })`**: muta las opciones serializadas de la entry, de modo que la siguiente reaplicación del include restaura `disabled: true` y desmonta la fila a mitad de sesión.
- **SIGTERM 143 para superficies de un solo uso mediante un handler de señal registrado por la app**: el propio handler del launcher compite con él por el código de salida; ganar esa carrera necesita una interfaz nueva de launcher, que es el coste que este cambio elimina.
- **Conservar `--dev` con la fila creada en runtime**: un estado intermedio de este cambio; aún necesitaba un fork de modo en el contrato de prompt, una variable `DSH_WEB_MODE` y arbitraje entre creación y fila de usuario, todo para evitar un poll ocioso cuyo coste es despreciable.

## Consecuencias

- Un despliegue que supervisa `dsh --profile headless` con SIGTERM ahora observa salida 0 en lugar de 143; el llamador envió la señal y no ve respuesta en stdout.
- La cadena de recarga se ejecuta en cada proceso de `dsh web`; un despliegue que no deba exponer `/plugins/events` deshabilita la fila `client-hmr` en su capa de parche.
- Las ejecuciones de un solo uso montan las filas de vigilancia de configuración que antes se saltaban, costando unos milisegundos de arranque.
- La divergencia vendida Loader/Include se reduce en un símbolo de protocolo y una máquina de estados, y `rescope-vendor:check` vuelve a pasar (la entrada de rescope del log de modificaciones se restaura a la posición que exige su ancla de edición exacta).
