# AGENTS.md

[English](AGENTS.md) | Español

Este directorio construye `landlock-run`, un lanzador de Landlock que se autolimita y luego ejecuta (self-restrict-then-exec): un binario de confinamiento pequeño y auditable distribuido como paquetes npm precompilados por plataforma, más el paquete de entrada JS ligero que lo resuelve e implementa su contrato CLI. Pertenece al workspace y lockfile pnpm raíz del repositorio. El repositorio principal es dueño de la CI nativa, el ensamblado de tarballs, la verificación y la publicación npm; mantén los cambios de la familia de paquetes coordinados con los consumidores del harness en el mismo repositorio.

## Postura previa al release

El proyecto está pre-1.0. Prefiere la API pública correcta sobre los parches de compatibilidad: si un nombre de paquete, un campo exportado, la estructura o un detalle del contrato está mal, renómbralo y actualiza todas las referencias en el mismo cambio. No añadas alias deprecados salvo que un release estable ya los necesite.

## Reglas de seguridad en runtime

- Toda herramienta debe fallar en modo cerrado (fail closed). Si no se puede crear un ruleset o el kernel no lo aplica, sale con código distinto de cero SIN ejecutar el comando envuelto. Nunca corras sin confinar como fallback.
- Los binarios de runtime y los paquetes de entrada NO aceptan sobrescrituras por variables de entorno: qué binario confina un proceso no debe ser nunca decidible por el entorno ambiente. La inyección de prueba es por parámetro de función; el prefijo `NALR_*` es solo para orquestación de build/test.
- La UAPI del kernel está autodefinida en el código C (verbatim de las cabeceras del kernel), lo que mantiene las compilaciones independientes de la antigüedad de las cabeceras del toolchain y hace de las definiciones parte del registro de auditoría.
- Ninguna biblioteca más allá de libc, enlazada estáticamente contra musl. La superficie de auditoría de una herramienta es su código C más el contrato estable de llamadas al sistema del kernel.
- El contrato CLI de cada herramienta ([docs/cli-contract.md](docs/cli-contract.es.md)) es el contrato de compatibilidad entre repositorios: la gramática de argv, los códigos de salida y las líneas de reporte cambian solo con un incremento de versión y una entrada de changelog, y los consumidores los parsean solo a través del paquete de entrada.
- Deliberadamente NO hay fallback de compilación en la instalación: un host sin un paquete de plataforma que coincida obtiene una ruta de lanzador inexistente, el probe del consumidor falla y el consumidor cae en modo cerrado — esa degradación es parte del diseño, no un hueco que rellenar con node-gyp.

## Estructura del repositorio

```text
packages/entry/     Published entry package: JavaScript API (resolve/probe/grants) + the C source.
packages/linux-*/   Published per-platform packages: one prebuilt static binary, no JavaScript.
scripts/            Build, matrix derivation, prepack gates, and release orchestration.
test/               Plain-node behavioral tests (entry API + real-kernel launcher proofs).
docs/               Architecture, packaging, CLI contract, release, support matrix, naming.
```

## Comandos

```sh
pnpm install
pnpm build:ts        # entry packages → lib/
pnpm build:native    # this Linux architecture's binaries (needs musl-tools); fails fast elsewhere
pnpm typecheck
pnpm test            # entry tests everywhere; launcher tests need linux + built binary
```

## Invariantes de empaquetado

- La matriz de paquetes es metadatos explícitos y verificados en el repo: `packages/<name>/package.json` (`os`, `cpu`), `packages/<name>/prebuilds.json` (los binarios que pueden existir allí) y [docs/support-matrix.md](docs/support-matrix.md) permanecen sincronizados cuando la matriz cambia. `scripts/github-matrix.mjs` deriva de ella las matrices de CI y release; nada más enumera plataformas.
- Los nombres de los paquetes de plataforma contienen solo la plataforma (`-linux-x64`), nunca variantes de herramienta — esas quedan dentro de `prebuilds.json`. El enlazado estático con musl es la razón de que no haya sufijo de libc: un binario sirve a las distros glibc y musl.
- Los paquetes de plataforma no llevan JavaScript; el paquete de entrada los resuelve a rutas de archivo. Los backends se demuestran a sí mismos en runtime mediante el probe funcional, nunca por confianza en los metadatos.
- Las compilaciones son solo nativas: cada arquitectura compila su propio binario en su propio runner (la CI es la compiladora de referencia); ningún cross toolchain entra en el repo.
- Cada tarball pasa compuertas en el momento del pack: los paquetes de plataforma se niegan a empaquetar sin sus binarios declarados presentes, ejecutables y en la arquitectura ELF correcta (`verify-launcher-binary.mjs`), los paquetes de entrada sin `lib/` compilado (`verify-entry-lib.mjs`), y el pipeline de release fija byte a byte los binarios instalados contra las compilaciones del workspace (`verify-packed-install.mjs`).
- Los tarballs de plataforma se empaquetan con `npm pack`, nunca con `pnpm pack`: la ruta de pack de pnpm elimina el bit de ejecución (observado en 11.7.0), entregando un lanzador que ningún consumidor puede lanzar. `pack-release.mjs` codifica esa separación; el ensayo comprueba la ejecutabilidad de la copia instalada para que una regresión falle ruidosamente en lugar de hacerse pasar por un kernel que no aplica la restricción.
- Los artefactos generados quedan fuera de git: `packages/*/bin/`, `packages/*/lib/`, `dist/`, `.release/`, `*.tsbuildinfo`. Las reglas de ignore viven solo en el `.gitignore` RAÍZ — un archivo de ignore anidado en un paquete puede dejar caer silenciosamente payload de los tarballs.

## Documentación

Los docs orientados al usuario están en inglés. Mantén el README centrado en la instalación, el uso y el estado de soporte; las decisiones de diseño duraderas pertenecen a docs/ junto al código, y la implementación actual pertenece a [docs/architecture.md](docs/architecture.es.md).
