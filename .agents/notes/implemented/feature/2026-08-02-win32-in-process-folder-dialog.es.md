# Agent Note: El selector de carpetas Win32 pasa a koffi en un proceso hijo

Status: implemented

[English](2026-08-02-win32-in-process-folder-dialog.md) | [中文](2026-08-02-win32-in-process-folder-dialog.zh.md) | Español

## Problema

El nivel principal del selector de directorios de Windows era un script PowerShell generado alrededor de `FolderBrowserDialog` de WinForms: el diálogo moderno solo donde PowerShell 7 resulta estar instalado, una regresión donde PowerShell 6 resuelve pero no tiene WinForms (la salida 1 no es `ENOENT`, así que el fallback 5.1 nunca se ejecutaba), un techo `SetProcessDPIAware` en el DPI del sistema, y un selector cuyo comportamiento dependía de qué shells trae una máquina en lugar de Windows mismo.

## Decisión

`packages/host/directory-picker-native` ahora abre `IFileOpenDialog` (`FOS_PICKFOLDERS | FOS_FORCEFILESYSTEM | FOS_NOCHANGEDIR`) en proceso a través de koffi — ya una dependencia del workspace para las otras superficies `win32.ts` del repo — como nivel win32 principal. La conversación COM se ejecuta en un proceso hijo generado para que el `Show` modal nunca bloquee el bucle de eventos del host; el hijo publica su id de hilo nativo antes de bloquearse, y el driver atiende los aborts re-publicando `WM_CLOSE` a las ventanas de ese hilo (`EnumThreadWindows`), matando al hijo cuando el presupuesto de cierre se agota. El diálogo es la primera ventana del hijo, así que Windows lo activa sin una llamada de foreground. El hilo hijo opta por el mejor reconocimiento de DPI de hilo que el host acepta (`SetThreadDpiAwarenessContext`, en cascada per-monitor-v2 → per-monitor → system-aware con el valor de retorno comprobado), una mejora estricta sobre el techo de DPI de sistema del script; el DPI sigue siendo un mejor esfuerzo cosmético — un host que no acepta ninguno recibe igualmente el diálogo moderno en lugar de una degradación. La división de módulos mantiene la cobertura honesta en cada host: `win32-dialog-logic.ts` (secuenciación pura) y `win32-dialog.ts` (driver) se prueban contra fakes en cualquier parte; `win32-dialog-bindings.ts` se prueba contra un mundo COM de `koffi` mockeado (la técnica de `dsh-session-persistence-jsonl`); los hosts POSIX ejecutan la plomería real de spawn hasta su rechazo de carga de koffi; los hosts win32 ejecutan un smoke real de abrir-y-abortar-cerrar. La cadena PowerShell que precedía a este nivel ha desaparecido (véase la [eliminación de la cadena](../simplification/2026-08-04-drop-windows-powershell-picker-fallback.es.md)): el nivel no tiene fallback.

## Alternativas consideradas

- **Un helper nativo precompilado (familia `native/` como `@deepseek-ai/node-addon-landlock-run`).** Rechazada: otra familia de paquetes npm, aprovisionamiento de MSVC y un carril de build/release de Windows — todo para publicar ~150 líneas de C que el repositorio no puede ejercitar hoy en CI (sin carril real de Windows); koffi entrega la misma superficie COM con cero cadena de suministro nueva.
- **Un addon N-API en proceso.** Rechazada por las mismas razones de CI/toolchain más C++ propio para el threading STA y el bombeo de mensajes que un proceso hijo + koffi expresan en TypeScript.
- **Mantener PowerShell principal y sondear versiones.** Rechazada: el selector sigue rehén del empaquetado de shells (6 vs 7, alias de Store, perfiles), y el diálogo heredado de 5.1 sigue siendo el suelo dondequiera que pwsh esté ausente; solo el ensanchamiento del disparador de fallback se aceptó en el nivel de fallback en su lugar.
- **Bloquear el hilo principal para la llamada modal.** Rechazada de plano: el host web debe seguir sirviendo RPC mientras el diálogo está abierto.

## Consecuencias

- Cada máquina Windows recibe el diálogo moderno con el mejor reconocimiento de DPI que soporta (per-monitor-v2 en 1703+), con PowerShell instalado o no.
- El renderizado real del diálogo y la ruta de selección siguen siendo una comprobación manual de Windows (el smoke de auto-cierre prueba abrir/anular/desenrollar).
- Las ranuras de la vtable COM y los GUID usados son ABI de Windows congelada (Vista); un error de firma de koffi arriesga una violación de acceso nativa, contenida al proceso hijo del diálogo — el proceso Node del host sobrevive y el fallo aflora tal cual (sin nivel de fallback; véase la [eliminación de la cadena](../simplification/2026-08-04-drop-windows-powershell-picker-fallback.es.md)). Los pins de ABI de koffi mockeado y el smoke win32 real existen para atrapar esos errores antes de publicar.
- El brazo del binario empaquetado — el ejecutable empaquetado generándose a sí mismo como entrada del diálogo — no lo ejercita ninguna prueba automatizada: el plano de fuente y el `lib/worker.cjs` compilado bajo node plano están cubiertos, y el spawn empaquetado queda diferido a la hoja de ruta de CI de Windows.
