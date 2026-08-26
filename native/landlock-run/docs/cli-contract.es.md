# Contrato CLI: landlock-run

[English](cli-contract.md) | Español

Este archivo fija el comportamiento observable externamente del lanzador — la superficie de compatibilidad entre repositorios entre los binarios y todo consumidor. Los consumidores interactúan con él a través del paquete de entrada (`launcherPath`/`probe`/`grantArgs`) y del protocolo del lanzador; cambiar cualquier cosa de lo que sigue exige un incremento de versión para toda la familia de paquetes y una nota en las notas de release.

## Gramática de invocación

```text
landlock-run [--ro <path>]... [--rw <path>]... -- <argv>...
landlock-run --probe
```

- `--ro <path>`: concede lectura + ejecución bajo `<path>`.
- `--rw <path>`: concede acceso completo al sistema de archivos bajo `<path>` (todo acceso que el ABI de kernel negociado pueda gobernar).
- Todo lo no concedido está denegado — los rulesets de Landlock son listas de permitidos.
- Un grant sobre un no-directorio conserva solo sus bits de acceso compatibles con archivos (así funciona un grant `--rw /dev/null`).
- `--`: separador obligatorio; todo lo que le sigue es el argv del comando, ejecutado mediante `execvp` con el entorno del lanzador sin cambios.
- `--probe`: mutuamente exclusivo con los grants y con un comando.
- Ninguna otra bandera, ninguna entrada por variables de entorno.

## Códigos de salida

- `125` (`LAUNCHER_FAILURE_EXIT`): todo fallo a nivel de lanzador — error de uso, kernel que no puede aplicar Landlock, root de grants que no se puede abrir, `exec` fallido. El comando envuelto NO se ejecutó.
- Tras un `exec` con éxito, todo estado de hijo se transmite sin cambios, incluido el 125. Los consumidores exigen por tanto tanto el estado 125 como una línea fatal `landlock-run: ` para atribuir un fallo del lanzador.
- `--probe`: `0` cuando el kernel aplica (total o parcialmente), `125` en caso contrario.

## Líneas de reporte

- El éxito del probe imprime exactamente una línea en stdout: `landlock: fully enforced` o `landlock: partially enforced (older ABI)`. El `probe()` del paquete de entrada las mapea a `full`/`partial`; una salida de probe distinta de cero se mapea a `unusable`.
- Una ejecución confinada bajo un kernel de ABI parcial imprime una línea en stderr `landlock-run: partial enforcement (older Landlock ABI)` y continúa — todavía confinada para todo lo que el kernel soporta.
- Todo error fatal imprime una línea en stderr con prefijo `landlock-run: ` antes de salir con `125`.

## Semántica de confinamiento

El lanzador establece `no_new_privs`, instala el ruleset sobre sí mismo y ejecuta (`exec`) el comando; el ruleset se hereda a través de `execve`, así que todo proceso descendiente queda igualmente confinado. El ruleset gobierna los accesos al sistema de archivos del ABI de Landlock negociado por el kernel (hasta el ABI 5); los accesos más nuevos que el ABI en ejecución no están gobernados y son la diferencia entre `full` y `partial`.
