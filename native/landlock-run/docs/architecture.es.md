# Arquitectura

[English](architecture.md) | Español

Este repositorio es dueño del *mecanismo* de confinamiento, no de la política: los consumidores (agent harnesses y capacidades de sandbox) deciden qué rutas puede leer o escribir una ejecución; esta familia de paquetes provee el lanzador que aplica esos grants y la API JavaScript que lo resuelve y le habla. El empaquetado sigue el modelo de paquete por plataforma de [`node-addon-require-builtin`](https://www.npmjs.com/package/@esplus/node-addon-require-builtin) (y esbuild), adaptado de los addons de Node a ejecutables estáticos independientes.

## Familia de paquetes en dos capas

La familia es un paquete de entrada más paquetes binarios por plataforma:

- **Paquete de entrada** (`@deepseek-ai/node-addon-landlock-run`): JavaScript ESM. Es dueño del contrato CLI de la herramienta — resolución de ruta (`launcherPath`), el probe funcional (`probe`), la construcción del argv de grants (`grantArgs`) y las constantes del contrato. Lleva el código C en su tarball para auditabilidad. Lista cada paquete de plataforma como `optionalDependency`.
- **Paquetes de plataforma** (`@deepseek-ai/node-addon-landlock-run-linux-{x64,arm64}`): un binario estático precompilado bajo `bin/`, un `prebuilds.json` que lo declara y nada de JavaScript. Los campos `os`/`cpu` de npm seleccionan el que coincide en la instalación; el paquete de entrada lo resuelve a una ruta de archivo — no hay nada que importar.

Como el parser CLI y el binario se versionan juntos en una sola familia de paquetes, el parser no puede quedarse atrás de la versión del binario. Evitar ese desajuste es la razón de que exista la división de paquetes.

No hay un paquete loader compartido: los paquetes de plataforma no tienen nada que cargar. Si alguna vez una segunda herramienta necesita JS compartido, extráelo entonces, no de forma preventiva.

## Resolución y disponibilidad

`launcherPath()` resuelve `@deepseek-ai/node-addon-landlock-run-<platform>-<arch>` y devuelve `<package>/bin/landlock-run`. Cuando el paquete no es resoluble devuelve una ruta de fallback determinista dentro del `node_modules` propio del paquete de entrada que simplemente nunca existe. La existencia está deliberadamente sin comprobar en ambos casos: `probe()` es la única señal de disponibilidad, y un binario ausente probea `unusable` exactamente igual que un kernel que no aplica la restricción. Los consumidores tienen una sola vía de degradación, no dos.

El probe es funcional — el lanzador construye y aplica un ruleset máximo real en un hijo de vida corta — porque las comprobaciones de versión pasarían por alto un kernel que tiene las llamadas al sistema pero se niega a aplicarlas.

## Fallo cerrado en todas partes

El lanzador sale con `125` sin ejecutar el comando ante cualquier fallo a nivel de lanzador: error de uso, kernel que no aplica la restricción, root de grants que no se puede abrir, exec fallido. La aplicación parcial (un ABI de Landlock más antiguo que gobierna solo un subconjunto de accesos) se acepta, se informa en stderr y el probe la expone como `partial` — el consumidor decide qué promete su vocabulario de modos en cada nivel. Ni el binario ni el paquete de entrada leen variables de entorno: qué binario confina un proceso no es nunca decidible por el entorno ambiente.

## Modelo de compilación y release

Las compilaciones son solo nativas. `scripts/build.ts` compila los binarios de la arquitectura en ejecución con el `musl-gcc` de la distro (estático: sin expectativas de loader ni libc en los consumidores, un binario para distros glibc y musl); los runners por arquitectura de la CI son los compiladores de referencia, y no existe ningún cross toolchain en el repo. La revisión cubre el código C y el job de CI que compiló cada binario, aplicada por tres compuertas: el prepack de plataforma rechaza binarios ausentes o de ELF incorrecto, el prepack de entrada rechaza `lib/` sin compilar, y el pipeline de release fija byte a byte los binarios instalados contra las compilaciones del workspace desde las que se empaquetaron.

La matriz de paquetes es metadatos verificados en el repo (campos `prebuilds.json` + `os`/`cpu`); `scripts/github-matrix.mjs` deriva de ella las matrices de CI y Release, así que añadir una plataforma extiende la automatización sin editar workflows.

## Añadir una plataforma

Una plataforma nueva añade un paquete `packages/<platform>/` (`package.json` con `os`/`cpu`, `prebuilds.json`, README, LICENSE), una entrada de runner en `scripts/github-matrix.mjs` y una fila en [support-matrix.md](support-matrix.md) — añadidas solo junto con un runner nativo de GitHub que la compile y la demuestre (la regla de no cross toolchain). Los lanzadores hermanos para otros mecanismos de confinamiento pertenecen a sus propios repositorios sobre esta misma plantilla, no como segundas herramientas aquí.
