# Agent Note: Una sola familia de editores en los presets de propósito general

Status: implemented

[English](2026-08-10-default-presets-single-editor.md) | Español

## Problema

Los presets `standard`, `code` y `cordis` exponían a la vez las herramientas de sistema de archivos `read`/`write`/`edit` y `str_replace_editor`. Las dos interfaces se solapan para la inspección y edición ordinarias de archivos, de modo que cada petición cargaba un schema de herramienta adicional sin añadir una capacidad por defecto distinta. El preset `minimal` tiene un contrato de composición diferente: su roster exacto de dos herramientas incluye intencionadamente `str_replace_editor` junto al `bash` persistente.

## Decisión

Las configuraciones de los presets `standard`, `code` y `cordis` montan `dsh-tool-fs` y `dsh-tool-fs-search`, pero no montan `dsh-tool-str-replace-editor`. Code Mode por tanto omite `str_replace_editor` tanto de su registro como del SDK generado. El preset `minimal` sigue montando `dsh-tool-str-replace-editor`, y los despliegues o presets creados por el usuario pueden seguir montando el plugin explícitamente.

Esta decisión estrecha el roster de los presets en lugar de eliminar el paquete de la herramienta o su soporte del runtime de Python. La [decisión anterior de roster compartido](../feature/2026-07-31-even-out-shipped-tool-rosters.es.md) sigue siendo dueña de por qué las herramientas neutras a la superficie viven en la composición de presets; esta nota es dueña de la excepción del editor.

## Alternativas consideradas

**Mantener ambas interfaces de edición en los presets de propósito general.** Rechazada porque los schemas solapados visibles para el modelo aumentan la elección de herramienta sin aportar una operación por defecto separada.

**Eliminar `str_replace_editor` de toda composición publicada.** Rechazada porque el preset `minimal` expone intencionadamente ese schema como una de sus dos herramientas, y los despliegues explícitos siguen siendo consumidores válidos del plugin independiente.

## Consecuencias

Los agents de propósito general usan `read`, `write` y `edit` para las mutaciones del sistema de archivos, mientras que el agent minimal retiene `str_replace_editor`. Las pruebas de composición de presets fijan su ausencia en el roster estándar, el roster de Cordis y el SDK de Code Mode, y las aserciones de minimal siguen fijando su presencia.
