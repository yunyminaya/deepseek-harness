# Nomenclatura

[English](naming.md) | Español

## Paquetes npm

La familia de paquetes públicos pertenece al alcance `@deepseek-ai` y usa el prefijo de paquete `node-addon-landlock-run`; los paquetes de plataforma añaden solo la información de plataforma:

```text
@deepseek-ai/node-addon-landlock-run
@deepseek-ai/node-addon-landlock-run-<platform>
```

Los sufijos de plataforma no llevan componente libc (los binarios son musl estático) ni componente de variante — las variantes permanecen dentro de `prebuilds.json` y de los nombres de archivo de los binarios.

## Binarios

El ejecutable lanzador es `landlock-run`, distribuido en `bin/landlock-run` dentro de cada paquete de plataforma.

## Variables de entorno

El prefijo `NALR_` (Node Addon Landlock Run) está reservado para la orquestación de build/test:

```text
NALR_REQUIRE_LANDLOCK   test-only: an unenforcing kernel fails instead of skipping
```

Los binarios de runtime y los paquetes de entrada NO leen variables de entorno — una regla de seguridad de runtime ([AGENTS.md](../AGENTS.md)), no una convención de nomenclatura. No incluyas el alcance npm en los nombres de variables de entorno.

## Símbolos C

El lanzador es un único archivo C con enlazado estático; no hay namespace de símbolos exportados. Las constantes UAPI del kernel conservan sus nombres de kernel con el prefijo `LL_` donde se definen localmente.
