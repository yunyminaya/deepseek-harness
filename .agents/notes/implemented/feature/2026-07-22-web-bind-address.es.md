# Agent Note: Dirección de bind explícita para la web

Status: implemented

[English](2026-07-22-web-bind-address.md) | Español

## Problema

`dsh web` se enlaza a todas las interfaces de red incluso cuando su navegador se ejecuta en la misma máquina. El uso local expone por tanto un servidor de desarrollo sin autenticación sin una elección explícita del operador, mientras que el uso en contenedores remotos y con navegador en la LAN todavía necesita una forma compatible de aceptar conexiones fuera del loopback.

El carrier HTTP también oculta la dirección de bind dentro de `startWebServer()`, por lo que los shells alternativos no pueden declarar su propia política de red en el límite del paquete.

## Decisión

`dsh web` se enlaza a `127.0.0.1` por defecto. El CLI acepta `--host 0.0.0.0` como el modo explícito de todas las interfaces y rechaza otros valores para que sus modos de red sigan siendo un contrato pequeño y deliberado. El modo de todas las interfaces sigue imprimiendo la URL de loopback y, cuando está disponible, la primera URL IPv4 externa.

`WebServerOptions.host` es obligatorio. El carrier HTTP pasa ese valor a `node:http` sin aportar un respaldo, dejando que cada shell sea responsable de su política de bind. Los consumidores programáticos del carrier pueden seleccionar directamente otro hostname o dirección.

## Alternativas consideradas

**Mantener `0.0.0.0` como valor por defecto.** Rechazado porque el uso ordinario en la misma máquina no necesita alcance en toda la red y no debería adquirirlo implícitamente.

**Usar una bandera booleana de exposición.** Rechazada porque `--host 0.0.0.0` nombra directamente el comportamiento del socket resultante y coincide con la opción subyacente del servidor sin introducir un segundo término.

**Valor por defecto dentro de `startWebServer()`.** Rechazado porque el carrier tiene múltiples shells posibles y ninguna base para elegir su política de despliegue. Exigir `host` hace visible la elección en cada llamada de ensamblaje.

## Consecuencias

Los arranques locales de `dsh web` siguen siendo accesibles en `http://127.0.0.1:3080`; un navegador en otra máquina debe optar por entrar con `dsh web --host 0.0.0.0`. El CLI todavía no expone direcciones de interfaz personalizadas ni modos IPv6, mientras que los consumidores programáticos del carrier conservan esa flexibilidad. Las pruebas del servidor fijan el reenvío tanto de loopback como de todas las interfaces en el límite de escucha de Node, y el smoke de la web sigue ejercitando la ruta por defecto del CLI.
