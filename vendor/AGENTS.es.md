# AGENTS.md — Paquetes vendored

[English](AGENTS.md) | Español

Este directorio contiene copias vendored de la fuente del framework Cordis y de sus librerías base. Consulta `vendor/README.md` para el manifest, el registro de modificaciones locales y el procedimiento de sincronización con upstream.

**NO edites los archivos de `vendor/*/src/` a la ligera.** Cada divergencia local respecto a upstream debe registrarse exhaustivamente en `vendor/README.md` bajo «Local modifications». Los archivos `vendor/*/tsconfig.json` son la excepción — se regeneran para encajar en el build del monorepo y pueden tocarse por cambios de política de type-checking (p. ej. `noImplicitAny`).

Cuando los cambios sean inevitables, sigue el procedimiento de sincronización de `vendor/README.md`.
