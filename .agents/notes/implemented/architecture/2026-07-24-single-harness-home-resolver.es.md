# Agent Note: Un único resolver del home del harness

Status: implemented

[English](2026-07-24-single-harness-home-resolver.md) | Español

## Problema

El harness tenía dos convenciones incoherentes sobre «dónde viven los datos de usuario de DeepSeek Harness»:

- `@deepseek-ai/dsh-home` resolvía `configured ?? $DSH_HOME ?? ~/.dsh`.
- `@deepseek-ai/dsh-home-paths` distribuía un **segundo** `resolveDshHome` con la misma precedencia más expansión de tilde — un casi duplicado de `dsh-home` que ningún gate señalaba porque los dos vivían en paquetes distintos y ya habían divergido (solo uno expandía tildes).

Dos resolvers para el mismo hecho transversal significaban que no existía una única política de home.

## Decisión

Un único resolver posee el home del harness, en `@deepseek-ai/dsh-home-paths`, de raíz única:

```
explicit configured path  >  $DSH_HOME  >  ~/.dsh
```

Un `$DSH_HOME` vacío o con solo espacios se trata como no establecido; de lo contrario, `resolve('')` colocaría silenciosamente el home en el directorio de trabajo actual. El harness mantiene todos los datos de usuario bajo una única raíz; no hay división XDG de config/data/cache. `dshHomePath(...segments)` une los hijos propiedad del despliegue a esa raíz, y `dsh-app-boot` lo expone a las expresiones `!!js` de config del Loader antes de montar las entradas, de modo que las composiciones distribuidas derivan `sessions` y `storages` sin copiar el resolver. `dshHomeDisplay()` nombra simbólicamente una raíz resuelta para las rutas visibles al usuario — `~/.dsh` para el home por defecto, `$DSH_HOME` para cualquier home configurado — de modo que la etiqueta `AGENTS.md` global del usuario nunca filtra una ruta absoluta de la máquina. Sustituye a la comprobación a medida por defecto-vs-`$DSH_HOME` de agent-instructions.

`@deepseek-ai/dsh-home` se elimina. Sus tres importadores (`dsh-tool-bash`, `dsh-skill-filesystem`, `dsh-agent-spine-demo`) importan `resolveDshHome` desde `dsh-home-paths`.

`dsh-telemetry` y su política de home separada están ausentes bajo la [eliminación del toolchain de proyecto SDK](../simplification/2026-08-11-remove-sdk-project-toolchain.es.md), dejando este resolver como la única política de home.

## Alternativas consideradas

**Dejar las dos copias de `resolveDshHome` en su sitio.** Ya habían divergido (una expande tildes, la otra no) y codifican el mismo hecho transversal dos veces. La consolidación es el propósito de la capa `util/`; un resolver duplicado es un bug latente de divergencia.

**Adoptar XDG (honrar `$XDG_CONFIG_HOME`, o dividir config/data/cache en árboles separados).** Se consideró y se descartó a favor de una única raíz evidente. Una verdad de referencia única `$DSH_HOME || ~/.dsh` coincide con `~/.claude` / `~/.aws`, no exige reclasificar por tipo cada consumidor de `~/.dsh`, y no deja asimetrías de resolver que reconciliar.

## Consecuencias

- Un único hecho de home, un único resolver. `dsh-home-paths` es el único propietario; el grupo `util/` pierde el paquete `home`.
