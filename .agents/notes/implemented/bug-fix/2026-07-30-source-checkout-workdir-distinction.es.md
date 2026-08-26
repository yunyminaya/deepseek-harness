# Agent Note: Las rutas del checkout de fuente no definen directorios de trabajo

Status: implemented

English | [中文](2026-07-30-source-checkout-workdir-distinction.zh.md)

## Problema

La sección de prompt `harness:source` sigue la [decisión de ubicación de fuente](../../archived/feature/2026-07-21-dsh-system-prompt-source-path.md), pero su redacción original llamaba al checkout «tu propio código fuente» sin distinguir esa ruta del workspace de la sesión. En una configuración TUI normal que no enuncia `{{cwd}}` en su persona, esta puede ser la única ruta absoluta fija cerca del inicio del system prompt. DeepSeek V4 podía por tanto responder «¿cuál es el workdir?» con el checkout del harness en vez de determinar el directorio de trabajo actual de la sesión.

Una declaración en bloque de que el checkout no es el directorio de trabajo también sería falsa. `dsh meta` hace intencionalmente del checkout de fuente ambos valores.

## Decisión

La sección identifica la ruta como el «checkout de la implementación de DeepSeek Harness». Dice que la ubicación del checkout y el directorio de trabajo actual son valores separados que pueden diferir, prohíbe inferir el directorio de trabajo desde la ruta del checkout, dirige al modelo a usar `pwd`, y limita el propósito del checkout a inspeccionar o extender DSH mismo.

La derivación de la ruta, la propiedad global de `harness:source` y el ordenamiento `-99` permanecen sin cambio. Describir los valores como conceptualmente separados y no como siempre desiguales mantiene la instrucción precisa tanto en sesiones ordinarias de proyecto como en `dsh meta`.

## Verificación

El test unitario de `dsh-app-boot` fija el texto exacto y su orden. El smoke keyless PTY del CLI inspecciona el header de petición ensamblado. El snapshot TUI `source-checkout-workdir` monta la sección con `/opt/dsh-source`, pregunta «¿cuál es el workdir?» a través de un turno grabado de DeepSeek V4, y exige que la transcripción en replay ejecute `pwd` y reporte el workspace generado en vez del checkout.

## Alternativas consideradas

**Decir que el checkout jamás es el directorio de trabajo.** Descartado porque `dsh meta` deliberadamente los hace la misma ruta.

**Poner el directorio de trabajo actual en la sección de fuente global.** Descartado porque la sección de fuente es global al launcher mientras el directorio de trabajo pertenece a cada sesión; combinarlos duplicaría la propiedad de `cwd` del bucle y haría variar un hecho estable de fuente por agente.

**Quitar la ruta de fuente del prompt.** Descartado porque las herramientas auto-referenciales de DSH siguen necesitando una ubicación fiable del checkout cuando el launcher arranca desde un proyecto no relacionado.

## Consecuencias

El prompt es más largo y una pregunta directa por el directorio de trabajo puede gastar una llamada barata a `pwd`. A cambio, el modelo ya no trata la ruta de la implementación del harness como un workspace implícito de tarea, mientras el modo meta sigue siendo veraz cuando ambos valores coinciden.
