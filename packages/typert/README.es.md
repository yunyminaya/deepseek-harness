# Typert

[English](README.md) | Español

Typert separa el análisis de fuente, el almacenamiento de runtime y el descubrimiento del Loader.

| Paquete | Rol | Clave de Cordis |
|---|---|---|
| [`registry/`](registry/README.es.md) | Almacena la reflexión de paquetes de runtime y los schemas | `ctx.typert` |
| [`loader/`](loader/README.es.md) | Descubre entradas del Loader y registra artefactos de host generados | consume `ctx.loader` y `ctx.typert` |
| [`generator/`](generator/README.es.md) | Genera artefactos de runtime a partir de tipos de fuente | librería en tiempo de build |
