# Agent Note: Desenlazar los junctions de los fixtures antes de la eliminación recursiva

Status: implemented

[English](2026-08-12-unlink-fixture-junctions-before-delete.md) | Español

## Problema

Los fixtures de install-lefthook y translation-pairing enlazan mediante junction los directorios reales `scripts/`, `node_modules` y el paquete tsx del repositorio dentro de árboles de fixture, para que las sondas del instalador se resuelvan a través de ellos. La eliminación recursiva de Windows puede tratar un junction (un punto de repetición MOUNT_POINT) como un directorio y seguirlo hasta su destino; el `worktree remove` de Git hizo exactamente eso y borró el `scripts/` registrado del repositorio y el paquete tsx (la instrumentación del incidente fijó la eliminación en ese paso). Una limpieza de fixture que confía en su eliminador borra entonces las fuentes del propio repositorio en lugar del fixture.

## Decisión

`scripts/test-fixture-cleanup.ts` es dueño del desmontaje de fixtures seguro frente a junctions: `unlinkFixtureLinks` recorre un árbol y desenlaza todo punto de repetición antes de que `removeFixtureSafely` elimine el árbol ya sin enlaces (con reintentos de handles asíncronos de Windows). Cada `afterEach` afectado y el hook previo a `worktree remove` lo llaman. La regla general vive en `docs/defensive-patterns.md`: elimina las rutas con forma de enlace con unlink, y reserva el `rmSync` recursivo para directorios reales conocidos.

## Alternativas consideradas

**Confiar solo en la eliminación recursiva.** Rechazada: que un eliminador dado siga junctions depende de la herramienta y de la versión, y una ruta a través de `git worktree remove` ya destruyó archivos registrados; ninguna limpieza puede apostar el repositorio a ese comportamiento.

**Copiar en lugar de enlazar los directorios reales.** Rechazada: los fixtures existen para sondear las rutas reales del instalador a través de sus contenidos reales, así que las copias dejarían de ejercitar la frontera bajo prueba.

## Consecuencias

El desmontaje de fixtures ya no puede alcanzar las fuentes del repositorio a través de junctions. El recorrido extra es una pasada de lstat/unlink sobre árboles de fixture pequeños. El defecto destructor de datos tiene ahora su porqué duradero junto a la regla de defensive-patterns, y el helper es la ruta de desmontaje compartida para futuros fixtures con junctions.
