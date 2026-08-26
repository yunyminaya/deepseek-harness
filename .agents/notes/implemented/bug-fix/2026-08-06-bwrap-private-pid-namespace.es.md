# Agent Note: Aislar bwrap del namespace de PID del host

[English](2026-08-06-bwrap-private-pid-namespace.md) | Español

Estado: implementado

## Problema

El backend de bwrap montaba un `/proc` nuevo mientras conservaba el namespace de PID del host. Un comando confinado podía, por tanto, ver procesos del host y seguir enlaces mágicos de procfs como `/proc/<pid>/root`, `/proc/<pid>/fd` o `/proc/<pid>/cwd` hasta la vista de montaje de un proceso del host. Cuando los controles de acceso permitían seguir uno de esos enlaces, la ruta escapaba del bind de solo lectura de la raíz del host del perfil y de la lista de permitidos de `workspace-write`. Las restricciones de ptrace del host a veces bloqueaban la ruta, pero esos permisos dependientes del despliegue no eran una frontera de confinamiento.

La [decisión de sandbox](../feature/2026-07-06-sandbox.es.md) original dejaba deliberadamente sin cambios la visibilidad de procesos porque `SandboxMode` promete efectos sobre archivos, no aislamiento general de procesos. Los enlaces mágicos de procfs convierten la visibilidad de los procesos del host en parte de la frontera de efectos sobre archivos para bwrap, así que esa elección no puede preservar los modos prometidos.

## Decisión

Cada perfil de bwrap usa `--unshare-pid` y monta `/proc` para ese namespace privado. El comando confinado puede observar y controlar sus descendientes, mientras que los procesos del host y sus enlaces mágicos de procfs están ausentes. Bubblewrap suministra el proceso PID 1 del namespace para recolectar a los descendientes.

La sonda funcional de bwrap usa el mismo constructor de perfiles que los wraps reales. Un host que no puede crear el namespace de PID rechaza por tanto bwrap durante la selección y recurre a Landlock, en lugar de aceptar una sonda más débil y fallar más tarde.

Esto es un invariante del backend de bwrap, no una nueva promesa de `SandboxMode`. Landlock y Seatbelt siguen dejando sin cambios la visibilidad de procesos, y ningún backend restringe el acceso a la red.

## Alternativas consideradas

- **Enmascarar enlaces procfs seleccionados conservando la visibilidad de procesos del host.** Las entradas por proceso son dinámicas, y cubrir solo `root` dejaría cruces equivalentes a través de `fd`, `cwd`, `exe` y futuros enlaces mágicos. Una lista de bloqueo no puede establecer la frontera.
- **Confiar en las comprobaciones de propiedad de ptrace y procfs.** Su comportamiento depende de la configuración del kernel, de la configuración del contenedor, de las credenciales del proceso y de la capacidad de volcado. Los procesos del mismo usuario pueden ser alcanzables, así que estas comprobaciones son defensa en profundidad, no la autoridad del perfil.
- **Eliminar `/proc` por completo.** Las herramientas de proceso ordinarias y la gestión de descendientes esperan procfs. Un namespace de PID privado con su procfs correspondiente preserva esos mecanismos sin exponer procesos del host.

## Verificación

Las pruebas unitarias de perfiles fijan el desacoplamiento de PID en ambos modos confinados. Las pruebas con bwrap real verifican que ambos modos informan de una identidad de namespace de PID distinta de la del harness, rechazan una escritura a través de `/proc/1/root`, dejan ausente el objetivo del host y siguen permitiendo que el comando observe, termine y espere a su propio descendiente.

## Consecuencias

- Los comandos confinados con bwrap ya no inspeccionan ni señalan procesos del host, incluidos los procesos del mismo usuario.
- `read-only` y `workspace-write` ya no dependen de la política de acceso a procfs del host para impedir escapes del perfil de montaje.
- Los hosts sin namespace de PID utilizable seleccionan el siguiente backend Linux compatible mediante la escalera existente de fallo en modo cerrado.
- La garantía cambiada es de confinamiento del kernel, no de salida visible al modelo, protocolo o texto de transcript, así que el e2e con backend real es la vía de aceptación ensamblada y no hay cambios de instantáneas.
