# @deepseek-ai/node-addon-landlock-run-linux-arm64

[English](README.md) | Español

Lanzador Landlock `bin/landlock-run` precompilado para linux-arm64 — un binario musl estático compilado de forma nativa (sin toolchain cruzada) a partir del código C distribuido en [`@deepseek-ai/node-addon-landlock-run`](https://www.npmjs.com/package/@deepseek-ai/node-addon-landlock-run). Los campos `os`/`cpu` de npm seleccionan este paquete en el momento de la instalación; el paquete de entrada lo resuelve a una ruta de archivo — no lleva JavaScript y nunca se importa.

El binario está ignorado por git y viaja en el tarball npm mediante la lista `files`; el gate `prepack` se niega a empaquetar cuando falta o tiene una arquitectura ELF incorrecta, y el pipeline de release fija byte a byte el binario empaquetado contra el build de CI del que proviene. El enlazado estático musl significa un único binario para distribuciones glibc y musl por igual — de ahí que no haya sufijo libc en el nombre.

Hermano: `@deepseek-ai/node-addon-landlock-run-linux-x64`.
