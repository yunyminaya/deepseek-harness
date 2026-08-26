# Agent Note: Directorios de sesión agrupados por proyecto

Status: implemented

[English](2026-07-24-project-session-directories.md) | [中文](2026-07-24-project-session-directories.zh.md) | Español

## Problema

Una raíz de persistencia puede ser local a un proyecto, compartida por varios proyectos, temporal o centralizada. Los buckets de cwd con hash mantenían funcionales todos los despliegues, pero hacían difícil navegar una raíz compartida porque un desarrollador no podía reconocer un proyecto por su nombre de directorio.

Cada sesión JSONL ocupaba además un archivo directamente dentro del bucket del proyecto. Esa forma no tenía directorio de propiedad para artefactos de sesión adicionales como metadatos, adjuntos, archivos de spill o estado de coordinación.

## Decisión

El backend JSONL guarda las sesiones bajo una clave de proyecto legible y da a cada sesión su propio directorio:

```text
<configured-root>/
  --<normalized-cwd>--/
    <encoded-session-id>/
      session.jsonl.zstd
```

El modo raw usa `session.jsonl`, y las sesiones sin cwd usan `_no-cwd`. Los separadores de filesystem y de unidad se convierten en `-`, las unidades de código inseguras usan `~XXXX`, y el nombre legible está acotado para mantener el componente dentro de los límites del filesystem.

La clave de proyecto no tiene a propósito sufijo de hash. Sigue la convención legible habitual que usan los coding agents y mantiene la ruta de proyecto normalizada como nombre completo del directorio. La normalización es con pérdida: rutas como `/a/b-c` y `/a-b/c`, o rutas largas con el mismo prefijo retenido, comparten un directorio de proyecto. Sus ids de sesión distintos siguen seleccionando directorios de sesión separados; la reutilización del mismo id de sesión sigue siendo una colisión de almacenamiento y se rechaza.

Los filesystems que no distinguen mayúsculas también pueden hacer que claves de proyecto con distinto uso de mayúsculas apunten a un mismo directorio físico. La validación de identidad acepta esa ortografía alternativa solo cuando la canonización del filesystem resuelve las rutas descubierta y esperada al mismo transcript. Una ruta canónica distinta sigue siendo corrupción, así que los alias de mayúsculas no debilitan la comprobación de colisión de mismo id en los stores sensibles a mayúsculas.

La raíz configurada sigue siendo una elección de despliegue. El layout ni selecciona una raíz global ni exige que los proyectos compartan una. Cuando un despliegue centraliza el almacenamiento, las rutas de proyecto siguen siendo reconocibles; una raíz local al proyecto usa la misma estructura determinista.

El id de sesión codificado nombra un directorio de propiedad en lugar del propio transcript. `SessionPersistence.locate()` sigue devolviendo la ruta fija del transcript, conservando las semánticas de `transcript_path` del hook y de `DSH_SESSION_JSONL`. El descubrimiento ignora otras entradas dentro del directorio de sesión para que el backend pueda añadir artefactos propiedad de la sesión sin otro cambio de layout.

La materialización diferida sigue ligada al transcript: `create()` no hace E/S de filesystem, y el primer append crea los directorios de proyecto/sesión antes de la publicación segura frente a colisiones del transcript. Los directorios vacíos no se listan como sesiones. El backend rechaza los artefactos planos `<project>/<id>.jsonl*` con un error de layout explícito; el formato previo a la release no proporciona migración automática de datos.

## Alternativas consideradas

**Conservar los hashes opacos de cwd.** Esto conservaba los nombres cortos pero frustraba la navegación por ruta de proyecto solicitada cuando varios proyectos comparten una raíz de persistencia.

**Poner los archivos de sesión directamente en cada directorio de proyecto.** Coincidía con la organización básica de archivos de Claude Code y pi, pero no dejaba ningún límite de propiedad a nivel de sesión para artefactos futuros.

**Añadir un sufijo de hash resistente a colisiones.** Distingue las rutas cuyas formas normalizadas colisionan, pero hace que el nombre del directorio sea más que la ruta de proyecto normalizada. La convención elegida acepta la agrupación por proyecto con pérdida a cambio del nombre más simple y reconocible.

**Imponer una raíz centralizada.** Rechazado porque la colocación del almacenamiento pertenece a la configuración de despliegue. La agrupación por proyecto es útil cuando las raíces se comparten e inofensiva cuando no.

**Cargar tanto el layout plano como el de directorios.** Rechazado bajo la postura de no compatibilidad previa a la release. Un layout aceptado mantiene deterministas las comprobaciones de identidad y el descubrimiento.

## Consecuencias

Los stores compartidos pueden navegarse por nombres de proyecto reconocibles, mientras que las raíces locales y personalizadas conservan su libertad de configuración existente. Cada sesión tiene un directorio disponible para artefactos futuros propiedad del backend, y los consumidores existentes de transcripts siguen recibiendo una ruta de archivo.

Los nombres de directorio de proyecto son más largos que los antiguos hashes de cwd de 12 hex. Las rutas muy largas muestran solo un prefijo acotado. Mover un proyecto suele seleccionar un directorio distinto, pero cadenas de cwd distintas que se normalizan al mismo nombre comparten un directorio de proyecto por diseño.
