# sdk/ — controla runtimes de Harness desde otro proceso

[English](README.md) | Español

Este grupo contiene la pila de protocolo para controlar un runtime de Harness desde otro proceso. Los callers aportan el ejecutable del runtime y su `cordis.yml`; este grupo no crea, configura, compila ni lanza proyectos de desarrolladores. La [decisión del SDK de TypeScript](../../.agents/notes/implemented/feature/2026-07-27-typescript-sdk-and-sdk-subagent-backend.md) es la dueña del contrato del cliente, y la [eliminación del toolchain](../../.agents/notes/implemented/simplification/2026-08-11-remove-sdk-project-toolchain.md) es la dueña del límite de producto.

| Paquete | Rol |
|---|---|
| [`protocol/`](protocol/README.md) | Define el wire protocol del runtime del SDK |
| [`client/`](client/README.md) | Controla un runtime de Harness a través de la API de cliente TypeScript |
| [`server/`](server/README.md) | Sirve a los clientes SDK fuera de proceso sobre JSON-RPC por stdio |
