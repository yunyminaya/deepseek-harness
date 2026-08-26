# Recetario: añadir un paquete vendored

[English](adding-a-vendored-package.md) | Español

Cuando el harness necesita otro paquete Cordis de upstream (p. ej. `@cordisjs/plugin-http`), se incorpora **vendored** como fuente fijada bajo `vendor/`, no como dependencia npm — consulta [la decisión de vendoring](../../.agents/notes/implemented/process/2026-06-11-vendor-cordis-as-source.es.md) para el porqué. [vendor/README.md](../../vendor/README.md) cubre *actualizar* un paquete ya vendored; esta guía es la lista archivo por archivo para añadir uno **nuevo**. (Validada contra el conjunto vendored existente; si se desvía, corrígela aquí.)

## 1. Copia la fuente

```
vendor/<dir>/
  package.json     # from upstream; set "private": true, rescope the name, keep exports/type
  tsconfig.json    # extends ../../tsconfig.base.json (see configuration below)
  src/             # the upstream src/ verbatim
  README.md LICENSE # if upstream ships them
```

`tsconfig.json` refleja a los otros paquetes vendored — `rootDir: src`, `outDir: lib/types`, las relajaciones de estrictez que necesita el código de upstream y una entrada `references` por cada otro paquete vendored que importe:

```jsonc
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "rootDir": "src", "outDir": "lib/types",
    "noUncheckedIndexedAccess": false, "exactOptionalPropertyTypes": false,
    "noImplicitOverride": false, "noUnusedLocals": false, "noUnusedParameters": false
  },
  "include": ["src"],
  "references": [{ "path": "../cordis" }, { "path": "../cosmokit" }]
}
```

Invariantes de `package.json`: `"private": true` (los paquetes vendored nunca se publican), reajusta el ámbito del `name` ([mapeo](../rescope.es.md)) conservando el `version`/`exports`/`type` de upstream, apunta los metadatos de declaración a `lib/types`, publica las salidas de declaración `.d.ts` y `.d.ts.map`, y lista sus dependencias cordis en `peerDependencies` (coincidiendo con el manifest de upstream). Las dependencias transitivas de upstream deben estar a su vez vendored o ya presentes — hacer vendoring de un paquete suele significar hacer vendoring de su árbol de dependencias (p. ej. `@cordisjs/plugin-http` arrastra `@cordisjs/fetch-file`).

Los imports/exports relativos locales en la fuente TypeScript vendored usan especificadores `.ts` explícitos tras la copia. Esta es una diferencia de build local al repositorio respecto de upstream: `rewriteRelativeImportExtensions` emite imports de runtime `.js` mientras las declaraciones conservan especificadores `.ts` explícitos que los consumidores TypeScript de NodeNext/Node16 pueden resolver.

## 2. Regístralo en las configs raíz

| Archivo | Cambio |
|---|---|
| `tsconfig.base.json` | añade `"<npm-name>": ["./vendor/<dir>/src"]` a `paths` |
| `tsconfig.host.json` | añade `{ "path": "./vendor/<dir>" }` a `references` (antes de las entradas `packages/*`; el código vendored entra al grafo solo a través del aggregate host) |
| `vendor/README.md` | añade una fila de la tabla de manifest (dir, nombre npm, versión, repo de upstream, commit SHA) y registra cualquier modificación local |
| `scripts/publint-all.ts` | solo si el paquete vendored se publica desde aquí (las dependencias vendored normalmente no — omítelo) |

Cubierto automáticamente por globs — sin ediciones necesarias: workspaces del `package.json` raíz (`vendor/*`), `tsdown.config.ts`, `vitest.config.ts`, `.oxlintrc.json`. Un `vendor/<dir>/tsdown.config.ts` por paquete solo es necesario SI la configuración de build difiere del predeterminado raíz (ESM/CJS dual o múltiples entradas — consulta `vendor/schemastery` y `vendor/logger-console`); su entrada debe leer el JS emitido bajo `lib/types`.

## 3. Cuida el guardián de manifest

`scripts/check-vendor-manifest.sh` (un hook de pre-commit) falla si algo bajo `vendor/*/src` está en stage sin que `vendor/README.md` también lo esté. Pon en stage la actualización del manifest junto con la fuente para que el commit pase.

## 4. Verifica

```sh
pnpm install        # registers the workspace
pnpm run typecheck
pnpm run build && pnpm run constraints
```

Ejecuta los checks de comportamiento que selecciona la [política de pruebas](../testing.es.md). El mapa `paths` de fuentes vive una sola vez en `tsconfig.base.json` y sirve a todos los grafos. El límite de aislamiento importante es el grafo de project references: la fuente vendored debe referenciarse a través de su propio `vendor/<dir>/tsconfig.json`, no arrastrarse al programa estricto de un aggregate ([layout](../development.es.md#typescript-project-layout)).
