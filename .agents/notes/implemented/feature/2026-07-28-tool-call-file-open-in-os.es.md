# Agent Note: Apertura del archivo de la llamada de herramienta en el SO

Status: implemented

[English](2026-07-28-tool-call-file-open-in-os.md) | Español

## Problema

Las filas de herramientas del chat trataban toda la línea de resumen como un objetivo de clic que abría el panel de detalles de la derecha, con un fondo de hover en la fila. Para las herramientas de sistema de archivos, la acción útil es abrir el archivo mencionado en la aplicación por defecto del sistema operativo, no inspeccionar el payload de herramienta crudo en una barra lateral.

## Decisión

Los resúmenes de ruta de las herramientas de archivo (argumentos `read` / `write` / `edit` que llevan `path` o `file_path`) se renderizan como enlaces subrayados en reposo con cursor de puntero. Hacer clic en la ruta llama a `host.openPath` a través de `WorkspaceRuntime.openPath`, resolviendo las rutas relativas contra el cwd de la sesión. Las filas de enlace de archivo deshabilitan el expand de args (el icono principal es inerte); el clic en toda la fila, el relleno de hover de la fila y el gesto de clic para abrir los detalles se eliminan de las filas de herramientas (incluidos los registros de bash y todo). El panel de detalles y su superficie de inyección permanecen para la selección programática; las filas ya no los impulsan.

`host.openPath` es un RPC unario privilegiado aceptado solo desde solicitudes de navegador loopback y del mismo origen (la misma protección de carrier que `host.pickDirectory`). Los adaptadores de plataforma abren sin shell: `open` en macOS, PowerShell `Invoke-Item` en Windows y `xdg-open` en Linux de escritorio; los documentos renderizables por el navegador prefieren el navegador por defecto nombrado en macOS y en Linux de escritorio. WSL es una forma de host separada aunque Node reporte `linux`: el adaptador reconoce su entorno o la versión del kernel de Microsoft, traduce la ruta Linux con `wslpath -w` y pasa la ruta Windows/UNC resultante al mismo handoff de PowerShell. Los datos de plataforma del opener y el runner de comandos son inyectables para las pruebas. Los argumentos de lectura solo URL (`web_fetch`) no son enlaces de archivo.

## Alternativas consideradas

- Mantener los detalles de clic en fila y añadir un affordance de archivo separado — rechazado; la petición de producto reemplaza el gesto de fila por el enlace de archivo.
- Abrir los archivos dentro de una vista previa en la app — rechazado; la petición es la aplicación por defecto del SO.
- Tratar WSL como Linux de escritorio — rechazado; un proceso WSL reporta `linux`, pero una asociación de escritorio Linux es opcional mientras su escritorio y navegador de operador ordinarios viven en Windows.
- Reutilizar la exención de timeout de `host.pickDirectory` — innecesario; el handoff de apertura de ruta se completa rápido bajo el deadline unario normal.

## Consecuencias

Hacer clic en una ruta de archivo de una fila de herramientas abre esa ruta en el host. Las filas de herramientas que no son de archivo son resúmenes inertes (los alternadores de expand permanecen donde la fila ya los soportaba). Los clientes remotos o no loopback no pueden invocar `host.openPath`. Un rechazo del Host o del SO es propiedad de la vista de chat: muestra el motivo lanzado y reintenta la misma ruta ([fallo de apertura de archivo](../bug-fix/2026-08-18-tool-row-file-open-failure.es.md)).

## Riesgos

- Los hosts de Linux de escritorio sin `xdg-open`, y los hosts WSL sin interop de Windows funcional (`wslpath` más `powershell.exe`), fallan el RPC; la vista de chat muestra ese error del Host y ofrece reintentar.
- Las rutas relativas sin cwd de sesión se reenvían verbatim y pueden fallar en el host.
