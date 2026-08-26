# Release

[English](release.md) | Español

Pre-1.0: trátalo como una lista de verificación de release, no como una política de estabilidad.

## Versionado

La raíz del workspace del lanzador y sus tres paquetes públicos comparten una única versión. Ejecuta el asistente de incremento desde la raíz del repositorio:

```sh
pnpm --dir native/landlock-run release:bump patch          # or minor / major / x.y.z
```

Actualiza `native/landlock-run/package.json` y todos los manifests de `native/landlock-run/packages/*`, refresca el lockfile de la raíz del repositorio (`--ignore-scripts --lockfile-only`) y ejecuta `release:verify`. Las versiones explícitas aceptan semver completo, incluidas las pre-releases (`pnpm --dir native/landlock-run release:bump 0.0.0-test.0`); el workflow de publicación pone las versiones pre-release bajo el dist-tag `next`, de modo que `latest` nunca apunta a un build de prueba. Mantén las dependencias `workspace:*` en la fuente; pnpm las convierte en versiones concretas durante el pack.

Los incrementos de versión son cambios de fuente normales: abre un PR de release (o un commit) con los manifests del lanzador y el lockfile de la raíz, fúndelo y crea después la etiqueta `landlock-run-vX.Y.Z` correspondiente desde ese commit. El namespace evita colisionar con las etiquetas de release de otras familias de paquetes del repositorio. El workflow de publicación valida que la etiqueta coincide con la versión de cada paquete del lanzador.

```sh
pnpm --dir native/landlock-run release:commit patch        # bump + stage + commit in one command
git tag landlock-run-v0.0.2
```

## Preflight

```sh
pnpm install --frozen-lockfile
pnpm --dir native/landlock-run build:ts
pnpm --dir native/landlock-run typecheck
pnpm --dir native/landlock-run test:entry
```

En un host Linux, ensaya también la vía de pack localmente:

```sh
pnpm --dir native/landlock-run build:native
pnpm --dir native/landlock-run test:launcher
node native/landlock-run/scripts/pack-release.mjs native/landlock-run/.release/npm --current-platform-only
node native/landlock-run/scripts/verify-packed-install.mjs native/landlock-run/.release/npm --current-platform-only
```

## Publicación

Usa el workflow `Landlock Run Release` del repositorio principal para que cada binario se compile en su runner nativo correspondiente:

1. Ejecútalo con `publish=false` (desde el commit de release) para compilar todos los binarios de plataforma, ensamblar y verificar los payloads, empaquetar los tarballs en orden de publicación, ensayar la instalación empaquetada y subir el artefacto `npm-tarballs` para inspección.
2. Crea y empuja la etiqueta `landlock-run-vX.Y.Z` que coincide con las versiones de los paquetes.
3. Ejecuta el mismo workflow desde esa etiqueta con `publish=true`.

El workflow publica solo desde los tarballs finales empaquetados, en el orden de `publish-order.txt` (los paquetes de plataforma antes que el de entrada que depende de ellos opcionalmente). Un ensayo de la plataforma actual puede consultar igualmente a npm por metadatos de un paquete de plataforma opcional incompatible; ese paquete no puede suministrar el lanzador del host, que proviene del tarball local correspondiente. Publicar cada paquete de plataforma antes que el de entrada garantiza que una versión pública del paquete de entrada nunca apunta por delante de sus paquetes de plataforma. El workflow soporta publicación de confianza de npm a través de GitHub OIDC; sin ella, proporciona un secreto `NPM_TOKEN` en el entorno `npm-publish`. Los paquetes se publican con `--access public`.

Los tres nombres de paquete con alcance deben inicializarse con un token de organización `@deepseek-ai` mediante el fallback `NPM_TOKEN`: npm [exige que un paquete exista antes de poder configurar un publicador de confianza](https://docs.npmjs.com/cli/v11/commands/npm-trust/). Después de que el primer release cree los tres paquetes, configura cada paquete para que confíe en `landlock-run-release.yml` de este repositorio con el entorno `npm-publish`, y retira el token de fallback cuando la política de la organización lo permita.

Fallback manual local (solo los paquetes de la plataforma actual) — siempre a través de `pack-release.mjs`, nunca `pnpm publish` directamente (la vía de pack de pnpm quita el bit de ejecutable del lanzador; consulta [packaging.md](packaging.es.md)):

```sh
node native/landlock-run/scripts/pack-release.mjs native/landlock-run/dist/npm --current-platform-only
node native/landlock-run/scripts/verify-packed-install.mjs native/landlock-run/dist/npm --current-platform-only
while IFS= read -r tarball; do npm publish "native/landlock-run/dist/npm/${tarball}" --access public; done < native/landlock-run/dist/npm/publish-order.txt
```

No hagas commit de archivos `.npmrc` con tokens ni overrides de registro.
