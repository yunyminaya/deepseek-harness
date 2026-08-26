# Agent Note: El loader interpola el campo `disabled` de la entry

Status: implemented

[English](2026-08-11-loader-entry-disabled-interpolation.md) | [中文](2026-08-11-loader-entry-disabled-interpolation.zh.md) | Español

## Problema

La capa de plataforma de Windows (entonces un `windows.cordis.patch.yml` separado junto al parche base, desde entonces plegado en las filas base — ver Decisión) deshabilitaba `tool-bash` en win32, pero los presets publicados montan cada uno una fila `tool-bash`. Las filas de preset componen al final, de modo que la fila del mismo id re-habilitaba la herramienta en Windows — la sesión tenía a la vez `tool-bash` (respaldado por PowerShell) y `tool-pwsh`, en silencio, porque ningún spec fijaba la capa de preset compuesta. Los metadatos de entry no tenían mecanismo condicional: `!!js` interpola solo bajo el `config` del plugin, y el [post-mortem 0002](../../../../docs/postmortem/0002-js-expression-disabled-filesystem-tools.es.md) documenta que `disabled: !!js ...` sigue siendo un objeto de expresión veraz, deshabilitando la fila en todas partes.

## Decisión

El loader interpola el campo `disabled` de la entry (`vendor/loader/src/config/entry.ts`): una expresión `!!js` se evalúa contra el contexto del loader en cada decisión de montaje. `disabled` es el único campo de metadatos interpolado; `id`, `name`, `group` e `inject` siguen siendo estáticos. El nodo crudo permanece en las opciones, de modo que el write-back conserva la forma `!!js`. Los presets publicados (standard, code, cordis) declaran ellos mismos las filas de herramienta de shell y las compuertan por plataforma — `tool-bash` con `disabled: !!js process.platform === 'win32'` y su gemelo `tool-pwsh` con la expresión invertida — de modo que la capa de preset expone exactamente una herramienta de shell por host; el overlay de web-app deshabilita las filas de host de ambas herramientas, dejando que el preset de cada sesión decida. `verify-cordis-config` ahora permite expresiones solo en `disabled`.

El mecanismo completa el pliegue de la capa de plataforma: el `cordis.patch.yml` del bundle base compuerta ambas pilas de shell en sus propias filas — `bash-sandbox`/`tool-bash` llevan `disabled: !!js process.platform === 'win32'`, y sus gemelos `pwsh-sandbox`/`tool-pwsh` se montan solo en win32 con la expresión invertida. La capa de plataforma de Windows separada del launcher (`windows.cordis.patch.yml` más `apps/cli/src/windows-shell.ts` y su inyección en boot, recomposición en vivo y dumps de configuración) se elimina — la capa existía solo porque los metadatos de entry eran estáticos, y con `disabled` interpolado la condición vive en la fila que gobierna.

## Alternativas consideradas

**Un campo declarativo `platform` en la fila.** Estático y comprobable por compuerta, pero un segundo mecanismo de composición junto a `!!js`, y la plataforma es solo la condición de hoy.

**Overlays de plataforma a nivel de preset.** Rechazado: la condición pertenece a la fila que gobierna — el mismo principio que pliega la capa de plataforma de Windows separada del launcher en las filas base.

## Consecuencias

Una fila puede compuertarse a sí misma por plataforma o entorno; una expresión mala falla ruidoso en el boot. Todo los demás campos de metadatos siguen siendo literales y la compuerta sigue rechazando expresiones allí — el peligro del post-mortem-0002 queda cerrado para `disabled` por evaluación, no por prohibición. El intercambio de shell de Windows pasó de una capa de parche inyectada por el launcher a las filas propias del bundle base: win32 monta la pila pwsh confinada, POSIX lleva las filas pwsh deshabilitadas, y un único archivo de parche compartido sirve a ambos listados — el mecanismo de capa del note de [pwsh por defecto en Windows](../feature/2026-08-01-windows-pwsh-default.es.md) queda superado. Las filas de herramienta de shell siguen la misma regla de un plano que cualquier otra fila declarada por preset: el overlay de web-app deshabilita las filas de host `tool-bash`/`tool-pwsh` y los presets declaran ambas con compuertas de plataforma invertidas, de modo que un preset puede retirar o reemplazar la herramienta de shell por sesión en cualquiera de los dos hosts. La pila PTY win32 ausente del preset `minimal` es un seguimiento de metadatos de preset.
