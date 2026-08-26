# Agent Note: Desenlazar enlaces de fallback de perfil obsoletos en lugar de rmSync

Status: implemented

[English](2026-08-12-unlink-stale-profile-fallback-links.md) | Español

## Problema

`healProfilesModuleFallback` repunta las entradas de `$DSH_HOME/profiles/node_modules` cuando una instalación se mueve, y los hosts de Windows conservan esas entradas como junctions. `ensureSymlink` eliminaba una entrada obsoleta con `rmSync(link)`, pero Node trata un junction como un directorio al eliminarlo: sin `recursive`, `rmSync` lanza `ERR_FS_EISDIR`, de modo que cada arranque desde una instalación movida o desde un segundo worktree se estrellaba antes de bootear. La prueba unitaria `replaces a wrong symlink` reproduce ese fallo en Windows exactamente en la llamada de eliminación.

## Decisión

`ensureSymlink` elimina un enlace obsoleto con `unlinkSync(link)`. `unlink` borra el reparse point o el symlink mismo en todas las plataformas y nunca desciende al destino, lo que preserva la garantía de fallo ruidoso de la función: nunca se elimina un directorio real. La [decisión de profile-plugin-bundles](../architecture/2026-08-05-profile-plugin-bundles.es.md) sigue siendo dueña de la resolución de dos anclas del fallback; esta nota solo es dueña de la primitiva de eliminación.

## Alternativas consideradas

**`rmSync(link, { recursive: true })`.** En Node 24 esto elimina el junction sin seguir su destino, pero `recursive` borraría silenciosamente un directorio real que reemplazara el enlace entre la guarda de `lstat` y la eliminación, debilitando el contrato de fallo ruidoso que motiva la guarda.

**`rmdirSync(link)`.** También elimina un junction en Windows, pero se lee como eliminación de directorio para un enlace, y `unlinkSync` es el modismo existente del repositorio para limpiar junctions.

**Eliminar y recrear cada entrada incondicionalmente.** Correcto, pero agita enlaces sin cambios en cada arranque y ensancha la ventana de carrera del heal concurrente.

## Consecuencias

Los arranques en Windows reparan instalaciones movidas o con segundo checkout en lugar de estrellarse con `ERR_FS_EISDIR`; el comportamiento POSIX no cambia porque `unlinkSync` también desenlaza symlinks ordinarios. La prueba existente `replaces a wrong symlink` ahora pasa en Windows donde antes reproducía el fallo. Dos healers concurrentes que eliminan el mismo enlace obsoleto siguen mostrando la segunda eliminación como `ENOENT`, sin cambio respecto a la implementación `rmSync` anterior.
