# @deepseek-ai/node-addon-landlock-run

[English](README.md) | Español

Lanzador Landlock que se auto-restringe y luego hace `exec` para confinar subprocesos en Linux: este paquete de entrada resuelve el binario precompilado por plataforma, ejecuta su sonda funcional de aplicación y construye su argv de concesiones — los consumidores nunca escriben las banderas del lanzador ni analizan su salida por sí mismos.

```js
import { grantArgs, launcherPath, probe } from '@deepseek-ai/node-addon-landlock-run';

const launcher = launcherPath();
if (probe(launcher) !== 'unusable') {
  const argv = [launcher, ...grantArgs({ readOnly: ['/'], readWrite: ['/tmp/work'] }), '--', 'bash', '-c', command];
}
```

El lanzador instala un conjunto de reglas Landlock sobre sí mismo y hace `exec` del comando envuelto; el conjunto de reglas se hereda a través de `execve`, de modo que todo el árbol de procesos queda confinado. Todo lo no concedido se deniega, y los fallos del lanzador salen con `125` sin ejecutar el comando — cierre ante fallos (fail-closed), nunca apertura ante fallos (fail-open). El contrato del binario está fijado en el `docs/cli-contract.md` del repositorio; el código C viaja en este tarball (`src/main.c`) para auditoría.

Paquetes de plataforma (dependencias opcionales seleccionadas por `os`/`cpu`, sin JavaScript dentro): `@deepseek-ai/node-addon-landlock-run-linux-x64`, `@deepseek-ai/node-addon-landlock-run-linux-arm64`. En hosts sin ninguno, `launcherPath()` devuelve una ruta determinista inexistente y `probe()` informa `'unusable'` — deliberadamente no hay fallback de compilación en el momento de la instalación.
