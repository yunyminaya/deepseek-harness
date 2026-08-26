# Agent Note: La instalación de Lefthook pasa a ser local al worktree

Status: implemented

[English](2026-07-27-worktree-local-lefthook.md) | [中文](2026-07-27-worktree-local-lefthook.zh.md) | Español

## Problema

Cada `pnpm install` ejecuta el [`postinstall`](../../../../package.json) raíz, cuyo [`install-lefthook.mjs`](../../../../scripts/install-lefthook.mjs) invoca `lefthook install --force`. Los worktrees enlazados de Git comparten de otro modo el directorio de hooks por defecto del repositorio común, así que una instalación en cualquier worktree puede reescribir los hooks que usa cualquier otro worktree.

Los hooks generados por Lefthook prefieren una ruta absoluta del binario capturada del worktree instalador antes de probar su fallback del worktree actual. Los hooks compartidos pueden por tanto ejecutar el binario fijado de otro worktree hasta que ese worktree desaparece, mientras que las instalaciones concurrentes escriben los mismos archivos.

## Decisión

La instalación de hooks tiene ámbito de worktree. Con `CI=true` o `GITHUB_ACTIONS=true`, el instalador retorna antes de cualquier descubrimiento o mutación de Git porque los trabajos automatizados no consumen hooks de colaborador. En caso contrario, exige Git 2.26 o superior para que `git config --show-scope` pueda reportar qué ámbito suministró un valor, actualiza un repositorio de formato 0 a formato 1, habilita `extensions.worktreeConfig` y asigna al worktree actual un `core.hooksPath` absoluto en `$GIT_DIR/dsh-hooks`.

Antes de actualizar el formato 0, el instalador rechaza `extensions.*` directo en la config común; también rechaza `core.worktree` o `core.bare=true` directos y configs de worktree dormidas no vacías que habilitar la extensión activaría. La migración elimina el `core.bare=false` directo porque false es el valor por defecto de Git. La config del repositorio común y cada `config.worktree` existente deben ser archivos regulares. Estas comprobaciones deshabilitan la expansión de include porque el analizador de formato de repositorio de Git también ignora los destinos incluidos. Un candado con ámbito de repositorio serializa la migración y las escrituras de hooks; su id de proceso, token aleatorio, identidad de archivo y contenido exacto deben seguir coincidiendo al liberarse. Los candados muertos o inválidos exigen recuperación manual antes que un rompimiento automático.

Cada directorio de hooks porta un marcador JSON de titularidad con la ruta absoluta publicada por última vez en la config del worktree. Tras el traslado de un checkout, ese marcador permite reemplazar únicamente el valor obsoleto titularizado exacto. Git siembra el `config.worktree` de un nuevo worktree enlazado desde el worktree principal; cuando esa semilla contiene la ruta reservada de hooks respaldada por marcador de un worktree registrado, el instalador reemplaza solo la config del nuevo worktree con su propia ruta. Antes de que corra Lefthook, el marcador y cada hook generado existente deben ser archivos regulares sin alias. El instalador resuelve el ámbito, origen y valor efectivos de `core.hooksPath`, incluidos los include activos de `config.worktree`; rechaza rutas de ámbito de comando, rutas de ámbito de worktree sin dueño y directorios reservados sin dueño. Una ruta heredada de sistema, global o de repositorio común exige `DSH_LEFTHOOK_ALLOW_HOOKS_PATH_OVERRIDE=1`, que inscribe solo el worktree actual en Lefthook. Los destinos `includeIf` inactivos no se inspeccionan recursivamente porque no afectan la configuración actual. La configuración de Git con ámbito de comando se elimina del entorno del subproceso de Lefthook tras la validación.

Si Lefthook falla tras cambiar `core.hooksPath`, el instalador restaura el valor previo del worktree; un fallo de reversión se reporta junto al fallo de instalación. Los archivos existentes en `$GIT_COMMON_DIR/hooks` nunca se eliminan ni reescriben. Pruebas enfocadas del instalador fijan el aislamiento, la configuración copiada del worktree nuevo, el rechazo de la migración, la titularidad y la reubicación, la instalación concurrente, las rutas personalizadas y la reversión.

## Alternativas consideradas

**Mantener los hooks generados compartidos y confiar en su fallback del worktree actual.** La ruta absoluta capturada gana mientras su worktree existe, así que el fallback no aporta aislamiento de versión ni de ciclo de vida.

**Apuntar cada worktree a un directorio `.githooks` consignado.** Un directorio relativo con seguimiento elimina las rutas absolutas generadas, pero cambiar el `core.hooksPath` compartido puede deshabilitar hooks en worktrees antiguos cuyas ramas no contienen ese directorio, y sigue acoplando cada worktree a un único valor de configuración compartido.

**Construir una capa gestora de hooks encadenados de propósito general.** El orden, el reenvío de argumentos, la semántica de fallo y las actualizaciones pasarían a ser comportamiento propiedad del repositorio, ajeno al aislamiento de Lefthook. El instalador, en cambio, rechaza rutas personalizadas específicas de worktree y hace explícita la anulación más estrecha de ruta heredada.

**Poner en lista blanca rutas de include de credenciales específicas del proveedor.** Los hooks de colaborador no se usan en CI, así que las exenciones de ruta acoplarían la seguridad del instalador a los internos de checkout del proveedor y debilitarían la validación estricta para las instalaciones de colaborador. El no-op de CI evita la mutación del repositorio sin ninguna exención.

**Dejar de instalar hooks automáticamente.** La configuración manual evita las escrituras compartidas, pero deja opcionales por accidente las comprobaciones baratas de commit y push del repositorio, sobre todo en worktrees de agent de vida corta.

## Consecuencias

Instalar o eliminar un worktree ya no cambia los hooks activos, la ruta del binario ni los bytes de hook generados de otro worktree. Las instalaciones concurrentes se serializan y la instalación repetida es idempotente, mientras que la frontera de trabajos y latencia propiedad de [Fast local Git hooks](2026-07-22-fast-local-git-hooks.es.md) queda sin cambios.

El repositorio pasa a ser un repositorio Git de formato 1 tras la primera instalación. El instalador exige Git 2.26 por `--show-scope`; la extensión de config por worktree es anterior a ese comando. Los gestores de hooks de worktree personalizados exigen una elección de integración explícita; las rutas de hooks heredadas pueden coexistir en otros worktrees, pero inscribir el worktree actual en Lefthook implica que esos hooks heredados no corren allí salvo que el colaborador los encadene a través de `lefthook.yml`.

Los hooks comunes heredados permanecen en disco para los worktrees no actualizados. Pueden quedar obsoletos, pero eliminarlos automáticamente rompería un worktree registrado cuya rama no ha adoptado este instalador.
