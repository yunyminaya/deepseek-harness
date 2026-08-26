# Reasignación de paquetes vendorizados

[English](rescope.md) | Español

El framework Cordis y sus bibliotecas base están vendorizados en [`vendor/`](../vendor/README.md) y se publican bajo el scope `@deepseek-ai`, porque cada paquete del harness declara el framework como peer dependency: publicar el harness publica esta capa con él y, con los nombres upstream, esa publicación les usurparía el nombre en el registro. Esta página es el mapeo de nombres; la decisión y sus consecuencias viven en el [Agent Note de rescope](../.agents/notes/implemented/process/2026-08-10-vendor-package-rescope.es.md), y los commits upstream en [`vendor/README.md`](../vendor/README.md).

## Mapeo de nombres

| Directorio | Nombre upstream | Nombre publicado | Versión | Rol |
|---|---|---|---|---|
| `vendor/cordis/` | `cordis` | `@deepseek-ai/cordis` | 4.0.0-rc.7 | Núcleo del framework: `Context`, `Service`, `Fiber`, eventos |
| `vendor/cosmokit/` | `cosmokit` | `@deepseek-ai/cosmokit` | 1.8.1 | Utilidades compartidas sobre las que se construyen el framework y Schemastery |
| `vendor/schemastery/` | `schemastery` | `@deepseek-ai/schemastery` | 3.18.0 | Schemas de configuración (`Schema`) detrás del `Config` de cada plugin |
| `vendor/loader/` | `@cordisjs/plugin-loader` | `@deepseek-ai/cordis-plugin-loader` | 1.0.0-rc.5 | Carga de `cordis.yml`, resolución de plugins, caché del repositorio |
| `vendor/include/` | `@cordisjs/plugin-include` | `@deepseek-ai/cordis-plugin-include` | 1.0.4 | Includes de configuración y overlays de patch |
| `vendor/group/` | `@cordisjs/plugin-group` | `@deepseek-ai/cordis-plugin-group` | 1.0.0 | Grupos de plugins anidados |
| `vendor/timer/` | `@cordisjs/plugin-timer` | `@deepseek-ai/cordis-plugin-timer` | 1.1.2 | Timers conscientes de la disposición en `ctx` |
| `vendor/hmr/` | `@cordisjs/plugin-hmr` | `@deepseek-ai/cordis-plugin-hmr` | 1.0.15 | Hot module replacement para plugins y configuración |
| `vendor/logger-console/` | `@cordisjs/plugin-logger-console` | `@deepseek-ai/cordis-plugin-logger-console` | 1.0.0 | Exportador de logger de consola |

Los subpath exports conservan su ruta: `@cordisjs/plugin-loader/repository` se convierte en `@deepseek-ai/cordis-plugin-loader/repository`.

## Lo que el renombrado no toca

- **Nombres de directorio y versiones.** `vendor/hmr/` sigue siendo `vendor/hmr/`, y cada paquete conserva la versión upstream que registra su fila de la tabla del manifest, de modo que el árbol vendorizado sigue leyéndose como una instantánea upstream.
- **Rangos de dependencia.** Una entrada de dependencia cambia su clave, nunca su rango: `"cordis": "^4.0.0-rc.7"` se convierte en `"@deepseek-ai/cordis": "^4.0.0-rc.7"`. `linkWorkspacePackages` resuelve esos rangos conservados a los workspaces fijados.
- **El prefijo integrado `cordis:` del Loader.** `cordis:include` y `cordis:group` son un prefijo de protocolo, no un nombre de paquete.
- **La familia de configuración `cordis.yml`**, incluidos `*.cordis.yml`, `*.cordis.snapshot.yml` y `cordis.patch.yml`.
- **Los paquetes del harness cuyos propios nombres contienen la palabra**, como `@deepseek-ai/dsh-tool-cordis`.
- **Los identificadores de runtime upstream**, como el `Symbol.for('schemastery')` de Schemastery y su campo de metadatos `vendor:`.
- **La prosa fuera de `docs/`.** `vendor/*/README.md`, los README de paquetes y los Agent Notes conservan los nombres con los que se escribieron; un `cordis` suelto allí también puede ser el nombre de una opción del SDK de Python o un id de agent-preset. Dentro de `docs/`, la prosa y todas las vallas de Markdown siguen el renombrado.

## Lo que tu código tiene que cambiar

| Sitio | Antes | Después |
|---|---|---|
| Import de módulo | `import { Context } from 'cordis'` | `import { Context } from '@deepseek-ai/cordis'` |
| Merge de eventos tipados | `declare module 'cordis'` | `declare module '@deepseek-ai/cordis'` |
| Clave de dependencia en `package.json` | `"@cordisjs/plugin-hmr": "^1.0.15"` | `"@deepseek-ai/cordis-plugin-hmr": "^1.0.15"` |
| Entrada de plugin en `cordis.yml` | `name: '@cordisjs/plugin-include'` | `name: '@deepseek-ai/cordis-plugin-include'` |

## Aplicación, verificación y reversión

[`scripts/rescope-vendor.ts`](../scripts/rescope-vendor.ts) es el dueño del mapeo anterior y realiza el renombrado, así que ninguna referencia se renombra a mano:

```sh
pnpm run rescope-vendor            # report what would change
pnpm run rescope-vendor --apply    # rewrite every reference
pnpm run rescope-vendor:check      # assert the post-state; runs in the hygiene gate
pnpm run rescope-vendor --apply --reverse   # return to the upstream names
```

Vuelve a aplicarlo tras una sincronización upstream ([procedimiento](../vendor/README.md)) y continúa con la regeneración que imprime: `pnpm install` para el lockfile, `pnpm run gen-third-party-notices` y `pnpm run verify-translation-pairing --write` para los pares bilingües que tocó.
