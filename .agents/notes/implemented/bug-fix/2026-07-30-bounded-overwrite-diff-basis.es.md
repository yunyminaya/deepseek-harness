# Agent Note: Bases de diff contextual de sobrescritura acotadas en el proveedor

Status: implemented

English | [中文](2026-07-30-bounded-overwrite-diff-basis.zh.md)

## Problema

`dsh-fs-local` devolvía el archivo previo completo en `FsWriteOutcome.before` para que los consumidores pudieran construir un diff contextual de sobrescritura. Esa pre-lectura solo-de-presentación estaba sin acotar: una sobrescritura grande podía asignar el archivo previo entero, y comprobar un stat de ruta anterior solo no podía imponer un límite porque un proceso externo podía reemplazar o hacer crecer el archivo entre el stat y la lectura. Un reemplazo grande además hacía que el hunk contextual se acercara al tamaño del reemplazo incluso cuando el archivo previo era pequeño. Esto cierra el límite diferido registrado por [result-time applied-hunk diffs](../../archived/architecture/2026-07-02-result-time-applied-hunk-diffs.md).

## Decisión

`LocalFileSystem.Config.diffBasisMaxBytes` es una configuración de despliegue: entero seguro positivo no mayor que los límites de asignación de Buffer y decodificación de cadenas del runtime, con 10 MiB por defecto. Una sobrescritura suministra `before` solo cuando el reemplazo UTF-8 está estrictamente por debajo de ese límite y el archivo previo abierto para la base también termina por debajo. La lectura previa abre un descriptor, comprueba ese descriptor y lee como mucho el conteo de bytes configurado en chunks conscientes de cancelación; alcanzar el límite devuelve `null`. Un cambio de tamaño tras el stat del descriptor también devuelve `null`, incluso si el tamaño final sigue bajo el límite, porque un prefijo parcial sería una base de diff incorrecta. Contenido previo binario o UTF-8 inválido igualmente devuelve `null`, como cualquier errno de fase de descriptor — un archivo previo borrado o vuelto ilegible entre el preflight del llamador y la apertura de la base no puede fallar una escritura que el llamador ya comprometió; solo la cancelación y las fallas non-errno se propagan. Estos resultados no bloquean la escritura atómica.

El proveedor local posee esta decisión porque `before` es su base opcional y best-effort: puede evitar adquirir contenido previo que el límite de par configurado ya ha vuelto inelegible. `tool-fs` sigue poseyendo el cómputo del diff, la retención y la presentación. La configuración es independiente de `tool-fs.readStreamMinSize`; el enrutamiento de lectura y la presentación de sobrescritura son políticas distintas y no necesitan compartir un valor.

`before: null` pide a los consumidores usar su fallback de archivo-completo existente. El límite acota solo la adquisición extra de contenido previo y la elegibilidad para un par contextual. No acota el reemplazo propiedad del llamador, el valor `after` devuelto, ni el renderizado de fallback de un consumidor.

## Alternativas consideradas

**Conservar un umbral hardcodeado igual al umbral de streaming de la herramienta de lectura.** Descartado porque el umbral de lectura es configurable por despliegue y propiedad del consumidor. Dos constantes del mismo valor crearían un acoplamiento cross-package sin imponer, mientras que la base de sobrescritura es en sí una elección de memoria/presentación del despliegue.

**Gatear solo el lado previo en el proveedor y topear el diffeo de contenido nuevo en `tool-fs`.** Descartado porque adquiriría texto previo incluso cuando el límite de par configurado del proveedor ya excluye el reemplazo, y partiría una regla de elegibilidad de `before` entre dos plugins. Los consumidores siguen libres de imponer límites adicionales de salida.

**Confiar en el tamaño del `probe()` inicial antes de usar una lectura ordinaria de archivo completo.** Descartado porque ese tamaño puede volverse stale antes de la lectura. El lector de descriptores debe imponer el límite sobre el objeto que realmente lee.

**Streamar un diff contextual para pares arbitrariamente grandes.** Descartado para este bug fix porque el seam actual de filesystem devuelve cadenas `before`/`after` completas y la implementación actual de diff las consume. Un diff en streaming exigiría un protocolo cross-package separado y un diseño de presentación.

## Consecuencias

Los despliegues pueden afinar el costo extra de la base de sobrescritura sin cambiar el enrutamiento de lectura. En o por encima del límite exclusivo, las sobrescrituras siguen teniendo éxito y permanecen visibles a través del fallback de archivo completo, pero pierden los hunks contextuales. Por debajo del límite, el proveedor puede aún sostener casi `diffBasisMaxBytes` de texto previo además del reemplazo del llamador. La lectura acotada de descriptor añade una secuencia open/stat/read para sobrescrituras elegibles, mientras evita que un probe de ruta stale convierta esa secuencia en una asignación sin límite.
