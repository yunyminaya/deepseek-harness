# @deepseek-ai/node-addon-landlock-run

[English](README.md) | Español

Un lanzador de [Landlock](https://landlock.io/) que se autolimita y luego ejecuta (self-restrict-then-exec) para confinar subprocesos en Linux, distribuido como paquetes npm precompilados por plataforma más un paquete de entrada JS ligero que resuelve el binario y habla su contrato CLI. Construido para agent harnesses (marcos de trabajo para agentes) y otros hosts que necesitan ejecutar comandos no confiables bajo una lista de permitidos del sistema de archivos sin confinarse a sí mismos.

La herramienta es **`landlock-run`** — un lanzador de [Landlock](https://landlock.io/) que se autolimita y luego ejecuta (self-restrict-then-exec) (~300 líneas de C11 sobre la UAPI cruda del kernel, enlazado estáticamente contra musl). Instala un ruleset de Landlock sobre sí mismo y ejecuta (`exec`) el comando envuelto; el ruleset se hereda a través de `execve`, así que el comando y todos los procesos que lance corren confinados mientras el proceso invocante permanece sin restricciones. Fail-closed (fallo cerrado): si el kernel no puede aplicarlo, sale sin ejecutar el comando.

## Instalación

```sh
npm install @deepseek-ai/node-addon-landlock-run
```

Los paquetes publicados usan un paquete de entrada más paquetes opcionales de plataforma:

```text
@deepseek-ai/node-addon-landlock-run
@deepseek-ai/node-addon-landlock-run-linux-x64
@deepseek-ai/node-addon-landlock-run-linux-arm64
```

Los campos `os`/`cpu` de npm hacen que los instaladores traigan solo el paquete de plataforma que coincide. No hay fallback de compilación en la instalación a propósito: en un host sin paquete de plataforma la ruta resuelta nunca existe, el probe informa `unusable` y el consumidor cae en modo cerrado.

## Uso

```js
import { grantArgs, launcherPath, probe } from '@deepseek-ai/node-addon-landlock-run';

const launcher = launcherPath();
if (probe(launcher) !== 'unusable') {
  const argv = [launcher, ...grantArgs({ readOnly: ['/'], readWrite: ['/tmp/work'] }), '--', 'bash', '-c', command];
  // spawn argv with your process runner of choice
}
```

La API pública es deliberadamente pequeña:

- `launcherPath()`: ruta absoluta del lanzador de este host (existencia deliberadamente no comprobada — el probe es la señal de disponibilidad).
- `probe(launcher?, { timeoutMs? })`: probe funcional de aplicación — `'full' | 'partial' | 'unusable'`.
- `grantArgs({ readOnly?, readWrite? })`: el argv de grants del lanzador; todo lo no concedido está denegado.
- `LAUNCHER_BIN` y `LAUNCHER_FAILURE_EXIT` (125): constantes del contrato. Un hijo ejecutado con éxito también puede devolver 125, así que los consumidores necesitan el diagnóstico fatal además del estado para atribuir un fallo del lanzador.

El contrato completo del binario (gramática de argv, códigos de salida, líneas de reporte) está fijado en [docs/cli-contract.md](docs/cli-contract.es.md).

## Soporte

linux-x64 y linux-arm64, kernel con Landlock habilitado (5.13+; el nivel de ABI determina la aplicación `full` frente a `partial` — ver [docs/support-matrix.md](docs/support-matrix.md)). Otras plataformas no tienen paquete deliberadamente: los consumidores ejecutan allí backends de confinamiento distintos.

## Desarrollo

```sh
corepack enable
pnpm install
pnpm build:ts        # entry packages → lib/
pnpm build:native    # this Linux architecture's binaries (apt-get install musl-tools)
pnpm test
```

Los binarios están en el gitignore y se compilan de forma nativa por arquitectura — localmente para tu propia máquina, por los runners por arquitectura de la CI como compiladores de referencia. Flujo de release: [docs/release.md](docs/release.md).
