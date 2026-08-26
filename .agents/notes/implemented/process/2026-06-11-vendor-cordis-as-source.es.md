# Agent Note: Vendor Cordis como fuente, no dependencias npm

Status: implemented

[English](2026-06-11-vendor-cordis-as-source.md) | [中文](2026-06-11-vendor-cordis-as-source.zh.md) | Español

## Problema

DeepSeek Harness está construido sobre el framework Cordis. Cordis core estaba en 4.0.0-rc.6 (un release candidate) cuando este repo empezó; el harness depende de internos del framework (ciclo de vida de las fibers, disposición de effects, despacho waterfall) cuyo comportamiento exacto importa a las garantías de corrección del agent loop.

## Decisión

Copiar los paquetes Cordis necesarios (core, loader, include, group, timer, hmr, logger-console) y las bibliotecas de fundación cordiverse (cosmokit, schemastery) a `vendor/` como fuente, aplanados, conservando sus nombres npm originales para que la resolución del workspace sea transparente. `pnpm-workspace.yaml` fija `linkWorkspacePackages: true`, así que los rangos semver coincidentes con upstream resuelven estos workspaces fijados tanto en ejecución de fuente como de artefacto construido. Las dependencias genuinamente de terceros (js-yaml, chokidar, @standard-schema/spec, …) permanecen en npm.

`vendor/README.md` es el manifest: repo upstream + commit SHA por paquete y un registro exhaustivo de modificaciones locales. Un guard de pre-commit (`scripts/check-vendor-manifest.sh`) rechaza cambios en la fuente vendorizada que no actualicen el manifest en el mismo commit.

## Alternativas consideradas

- **Depender de los paquetes npm** — rechazada: core estaba en un release candidate, y el harness se apoya en internos del framework (ciclo de vida de las fibers, disposición de effects, despacho waterfall) de cuyo comportamiento exacto dependen las garantías de corrección del agent loop; un bump de un RC upstream podría romperlas sin ruta de arreglo local.
- **Vendorizar todo lo transitivo** — rechazada: las dependencias genuinamente de terceros (js-yaml, chokidar, @standard-schema/spec, …) permanecen en npm; solo se posee la capa del framework cuyos internos importan.

## Consecuencias

- El harness posee por completo su capa de framework: auditable, parcheable, fijada — un RC upstream no puede rompernos, y podemos arreglar bugs del framework in-tree.
- Los paquetes construidos ejecutan la misma generación vendorizada de Cordis que las pruebas de fuente; eliminar el enlazado de workspaces sustituiría silenciosamente copias npm detrás de nombres de paquete sin cambios.
- La sincronización con upstream es manual (procedimiento documentado en el manifest). El registro de modificaciones mantiene conocida la superficie del diff.
- Los paquetes vendorizados conservan el estilo de código upstream; los gates de lint/estrictidad los excluyen (sus tsconfigs relajan localmente nuestros flags de compilador más nuevos).
- Existe un parche local desde el primer día: eliminados los imports locale-YAML de hmr (el hook de import YAML del runtime no está vendorizado).
