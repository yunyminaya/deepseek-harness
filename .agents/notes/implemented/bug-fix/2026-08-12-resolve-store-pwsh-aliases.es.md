# Agent Note: Resolver los alias de pwsh de Microsoft Store

Status: implemented

[English](2026-08-12-resolve-store-pwsh-aliases.md) | Español

## Problema

`resolvePwshPath` documentaba que las instalaciones desde Microsoft Store se resuelven a través de PATH, pero su sonda de existencia era `existsSync`, que hace stat de un candidato y por tanto sigue los puntos de repetición. El `%LOCALAPPDATA%\Microsoft\WindowsApps\pwsh.exe` de la Store es un alias de ejecución de aplicaciones cuyo directorio de destino rechaza el stat mediante su ACL (EACCES), de modo que `existsSync` no lo encontraba y la resolución caía en silencio hacia Windows PowerShell 5.1 en hosts cuyo único PowerShell 7 es una instalación de la Store.

## Decisión

`candidateExists` acepta un candidato cuyo stat indique un archivo o que `lstat` vea como un punto de repetición con forma de enlace, y `resolvePwshPath` lo usa. Hacer spawn de la ruta del alias funciona porque CreateProcess resuelve los alias de ejecución de aplicaciones. Se acepta un candidato con forma de enlace aunque su destino no exista, para que un pwsh roto falle de forma ruidosa en el spawn en lugar de degradarse en silencio hacia la 5.1.

## Alternativas consideradas

**Sondear directamente el directorio del paquete en WindowsApps.** La ruta del paquete de la Store está versionada y oculta por ACL; fijarla en el código duplica un conocimiento de empaquetado que PATH más el alias ya poseen.

**Mantener el fallback a 5.1 ante fallos de stat.** Rechazado: ejecuta en silencio un shell distinto del instalado, que es exactamente el defecto que esta nota corrige.

## Consecuencias

PowerShell 7 instalado desde la Store ahora se resuelve antes que el fallback a 5.1 en Windows; los candidatos que son archivos reales y el comportamiento fuera de Windows no cambian. La prueba unitaria del symlink colgado fija la separación stat/lstat en todas las plataformas.
