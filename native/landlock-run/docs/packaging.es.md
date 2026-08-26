# Empaquetado

[English](packaging.md) | Español

La familia de paquetes usa el mismo diseño que los paquetes nativos como esbuild: un paquete JS de entrada más los paquetes opcionales de plataforma. A diferencia de los addons de Node no hay división de ABI ni de backend — cada paquete de plataforma lleva exactamente los ejecutables estáticos que declara su `prebuilds.json`.

## Paquetes publicados

```text
@deepseek-ai/node-addon-landlock-run
@deepseek-ai/node-addon-landlock-run-linux-x64
@deepseek-ai/node-addon-landlock-run-linux-arm64
```

Las plataformas no soportadas están deliberadamente ausentes de `optionalDependencies` — consulta [support-matrix.md](support-matrix.es.md).

## Matriz de paquetes

La matriz es explícita en metadatos versionados:

- `packages/entry/package.json` lista los paquetes de plataforma como `optionalDependencies`.
- `packages/<name>/package.json` declara `os` y `cpu`. No hay campo `libc` a propósito: los binarios están enlazados estáticamente contra musl y funcionan igual en distribuciones glibc y musl.
- `packages/<name>/prebuilds.json` declara los binarios que pueden existir en ese paquete (`tool`, `kind`, `path`).
- [support-matrix.md](support-matrix.es.md) explica por qué los paquetes de plataforma no soportados no se publican.

`scripts/github-matrix.mjs` deriva de estos archivos las matrices de CI y de Release. `scripts/build.ts` compila solo los targets del host actual, en `packages/<name>/bin/`; no es un generador de matrices. Al cambiar la matriz, actualiza los metadatos de paquete, `prebuilds.json`, el lockfile y los docs de soporte/release en el mismo cambio.

## Selección en tiempo de ejecución

1. Los campos `os`/`cpu` de npm hacen que los instaladores descarguen solo el paquete de plataforma que coincide.
2. `launcherPath()` del paquete de entrada lo resuelve a `<package>/bin/landlock-run`; los paquetes no resolubles producen una ruta de fallback determinista que nunca existe.
3. `probe()` es la única señal de disponibilidad: binario ausente y kernel que no aplica Landlock son deliberadamente indistinguibles (`unusable`), de modo que los consumidores tienen una única vía de cierre ante fallos.

## Sin fallback de instalación

El paquete de entrada NO tiene script de instalación y nunca compila en el host del consumidor. Un fallback de compilación exigiría una toolchain de musl en todas partes y convertiría una degradación limpia de cierre ante fallos en un quizá dependiente del entorno. La verificación del manifest empaquetado en `verify-packed-install.mjs` hace cumplir la ausencia de scripts de ciclo de vida de instalación.

## Gates de empaquetado

Los tarballs de plataforma los produce `npm pack`; los tarballs de entrada, `pnpm pack` — una división deliberada: `pnpm pack` (observado en 11.7.0) normaliza los modos de archivo y quita el bit de ejecutable, lo que distribuiría un lanzador que ningún consumidor puede ejecutar, mientras que los paquetes de plataforma no tienen dependencias y por tanto no necesitan la conversión del protocolo de workspace de pnpm; los paquetes de entrada necesitan esa conversión y no llevan ejecutables. `scripts/pack-release.mjs` codifica la división — nunca empaquetes a mano un paquete de plataforma con pnpm.

Ambas vías de empaquetado producen los bytes exactos de publicación detrás de un gate `prepack`:

- Paquetes de plataforma: `scripts/verify-launcher-binary.mjs` — cada binario declarado presente, ejecutable, con el `e_machine` ELF que coincide con la `cpu` declarada, y nada no declarado en `bin/`.
- Paquetes de entrada: `scripts/verify-entry-lib.mjs` — el `lib/` compilado presente.

`scripts/verify-packed-install.mjs` ensaya después el camino del consumidor desde los tarballs empaquetados: comprobaciones de payload, una instalación desechable, una fijación byte a byte del binario instalado contra el build del workspace, una comprobación de ejecutabilidad de la copia instalada y una prueba de confinamiento real a través del lanzador instalado. Un binario no ejecutable o ausente falla ruidosamente aquí en lugar de hacerse pasar por un kernel que no aplica Landlock.
