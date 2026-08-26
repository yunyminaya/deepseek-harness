# Agent Note: Preservar las DACL de Windows durante el reemplazo atómico de archivos

Status: implemented

[English](2026-07-19-windows-atomic-write-dacl-preservation.md) | Español

## Problema

Las escrituras atómicas protegen los directorios de staging de POSIX con `0o700` y los archivos temporales con `0o600`, pero los mode bits de Windows solo exponen una vista sintética de solo lectura de la DACL real. Crear el staging bajo el directorio padre del destino y confiar en la herencia es suficiente para un archivo nuevo, pero no para reemplazar un archivo existente cuya DACL explícita o protegida es más estrecha que la de su padre: el contenido se escribe bajo la DACL más amplia del padre, y el rename traslada ese descriptor de staging al reemplazo.

## Decisión

`dsh-fs-local` lee la DACL de un destino existente con `GetFileSecurityW`, la aplica al archivo temporal vacío con la herencia protegida antes de escribir el contenido y publica el temporal cerrado con `ReplaceFileW`. El descriptor de staging protegido impide que las entradas heredadas del directorio temporal amplíen el acceso; `ReplaceFileW` conserva la política de acceso original del destino y los demás metadatos del reemplazo. Su fusión de ACL puede reserializar el estado de autoherencia o duplicar ACE equivalentes, así que los buffers de descriptor self-relative no son un contrato de igualdad estable. Los archivos nuevos de Windows no tienen un descriptor previo que conservar y siguen heredando la DACL del directorio de destino; por eso su directorio de staging vive junto al destino. POSIX mantiene los modos de staging de solo propietario y conserva el modo de un destino existente.

La cobertura nativa de Windows protege una DACL de destino, inspecciona el archivo de staging escrito y compara la política de ACE ordenada y deduplicada del reemplazo final. Las pruebas de binding independientes del host cubren la traducción de errores de Win32 y cada frontera de llamada nativa. Las aserciones de mode bits siguen siendo solo de POSIX; la herencia de DACL de un archivo nuevo es un contrato del sistema operativo, no una allowlist de cuentas específica de la máquina.

## Alternativas consideradas

**Confiar en la herencia del directorio para los reemplazos.** Rechazada porque un destino puede llevar una DACL explícita o protegida más estrecha que la de su padre, así que la herencia ni protege el contenido en staging ni conserva la política de acceso del destino.

**Usar `ReplaceFileW` sin proteger el temporal.** Rechazada porque repara el descriptor final solo después de que el contenido ya se ha escrito bajo la DACL heredada del archivo de staging.

**Instalar una DACL de solo propietario en cada escritura.** Rechazada porque descartaría el uso compartido deliberado del proyecto. Copiar la DACL del destino conserva la política de acceso existente del despliegue en lugar de inventar una.

**Afirmar las cuentas heredadas con `Get-Acl` o `icacls`.** Rechazada porque una prueba así verifica la política de la máquina, no el comportamiento del paquete, y los nombres localizados de las cuentas conocidas hacen la salida inestable entre hosts.

**Omitir las llamadas existentes a `chmod` en Windows.** Rechazada porque Node mapea estos modos de escritura a no-ops benignos; los guards de plataforma añadirían ramas sin cambiar el comportamiento de la DACL.

## Consecuencias

Reemplazar un archivo de Windows ahora exige permiso para leer la DACL del destino y establecer la DACL del temporal; el fallo es ruidoso antes de escribir el contenido. El paquete incorpora Koffi para las llamadas Win32 específicas, cargado solo en las rutas de reemplazo de Windows. Un archivo nuevo de Windows hereda el acceso amplio del directorio cuando el directorio es amplio por diseño, mientras que el contenido temporal de POSIX sigue siendo de solo propietario; un destino de Windows de solo lectura sigue fallando la publicación antes de que la reproducción sintética del mode pudiera importar.
