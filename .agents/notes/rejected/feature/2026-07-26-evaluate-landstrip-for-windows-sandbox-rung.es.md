# Agent Note: Evaluar landstrip antes de construir un launcher de sandbox para Windows

Status: rejected — landstrip no está probado en batalla (un proyecto de un solo mantenedor con pocos días de vida, ~48 estrellas de GitHub en el momento del rechazo); una dependencia que es un invariante de seguridad debe tener adopción demostrada, así que el eslabón win32 conserva el plan del launcher interno

[English](2026-07-26-evaluate-landstrip-for-windows-sandbox-rung.md) | Español

## Problema

La [decisión de sandbox](../../implemented/feature/2026-07-06-sandbox.es.md) deja `PLATFORM_CHAINS.win32` vacío y planea llenarlo con «un runner de confinamiento de la familia AppContainer/token restringido, distribuido desde su propio repositorio sobre la plantilla `node-addon-landlock-run`» — un repositorio nuevo de ~1,500 líneas estimadas (el subárbol landlock-run tiene ~1,460 líneas de C/TS/scripts/tests más docs y CI) escrito y mantenido internamente.

Desde que se escribió esa nota, ha aparecido un runner de terceros con mantenimiento: `@landstrip/landstrip` (npm, en desarrollo activo, núcleo Rust con `optionalDependencies` precompiladas por plataforma) cubre Landlock + seccomp en Linux, Seatbelt en macOS y AppContainer/usuario restringido en Windows, con entrada de política JSON/YAML y un canal de notificación de denegaciones por trap-fd. Se envuelve con exec como bwrap, así que encaja en la forma `confine(argv)` de la cadena sin tocar los eslabones de Linux/macOS.

## Propuesta

Cuando se retome la fase del sandbox de Windows, evaluar envolver el backend de Windows de landstrip como runner del eslabón `win32` antes de redactar un repositorio interno de launcher AppContainer. La evaluación debe responder:

- **Síntesis de sonda.** landstrip no tiene `--probe`; el contrato de sonda funcional de la cadena tendría que sintetizarse a partir de una ejecución con trap.
- **Mapeo de dialectos.** Los dialectos de stderr de denegación y de fallo del runner, y la clasificación de código de salida de cierre ante fallos (fail-closed), necesitan un mapeo explícito al vocabulario de la cadena.
- **Licencia.** Los binarios son LGPL-2.1-or-later; se requiere una revisión de distribución antes de que entre en el conjunto de artefactos que se distribuyen.
- **Registro de fuente y build.** Cada binario del launcher interno está fijado byte a byte a una compilación nativa de CI de un archivo C revisable de ~300 líneas; landstrip es un conjunto de binarios Rust de un solo mantenedor. Para el *eslabón Linux existente* ese intercambio ya está decidido — no lo cambies ([nota de sandbox](../../implemented/feature/2026-07-06-sandbox.es.md) y la migración propia del launcher fuera de una dependencia Rust). Para un eslabón que aún no hemos construido, sopesar el mantenimiento de terceros frente a un segundo repositorio nativo interno es una cuestión genuinamente abierta.

## Alternativas consideradas

- **Construir el launcher AppContainer interno como estaba planeado.** Sigue siendo la opción por defecto si la evaluación falla en licencia, auditabilidad de fuente/build o encaje de la sonda; el coste es ser dueños de un segundo launcher de seguridad nativo indefinidamente.
- **Cambiar también el eslabón Linux de Landlock a landstrip.** Rechazado de plano: la corrección del sandbox es un invariante de seguridad, el launcher actual tiene fuente C revisable y binarios fijados byte a byte a compilaciones nativas de CI, y ya migró fuera de una dependencia Rust precisamente por esta razón.

## Criterios de aceptación

- Antes de que comience cualquier implementación del eslabón Windows, una evaluación registra las respuestas de sonda, dialecto, licencia, repositorio de fuente, proceso de release y build de binarios, y el go/no-go se añade al plan de fases diferidas de la nota de sandbox.

## Riesgos

- Cadena de suministro de un solo mantenedor en una posición crítica de seguridad — la razón por la que esto es una compuerta de evaluación, no una decisión de adopción.
- El paquete es joven; su API y su empaquetado pueden cambiar antes de que empiece la fase de Windows, así que vuelve a verificarlo contra el registro en vivo entonces.
