# Agent Note: El escaneo inicial de HMR interbloqueó un boot fallido en un exit 13 silencioso

Status: implemented

English | [中文](2026-08-03-hmr-initial-scan-boot-deadlock.zh.md)

## Problema

Un lanzamiento de `dsh` cuyo árbol de config falló la validación salía con 13 (top-level await sin liquidar) sin ningún diagnóstico, y dejaba el estado de terminal del TUI varado en el shell — el síntoma exacto que el [release fail-loud](2026-07-31-fail-loud-releases-the-terminal.es.md) arregló, reintroducido por un mecanismo distinto tras la [recarga transaccional de config](2026-07-20-config-hot-reload-resilience.es.md).

Dos defectos se compusieron:

1. **Includes concurrentes aplicando corrompen la actualización transaccional del grupo.** El escaneo inicial de chokidar del watcher principal de HMR re-anuncia cada archivo existente como `add`. Su `add` del archivo de config disparó `Include.refresh()` mientras el apply inicial del Include seguía en vuelo (`this.content`, la clave de dedup de contenido-cambiado, se comite solo tras el apply). Dos llamadas `EntryGroup.update` concurrentes sobre un grupo intercalan create y rollback sobre las mismas entradas, y la fiber del Include jamás se liquida — `loader.create` cuelga, `boot()` ni resuelve ni rechaza, y Node sale 13 cuando el loop drene.
2. **Los applies serializados por sí solos interbloquean el rollback del fallo.** Con las mutaciones de Include encoladas, un apply inicial fallido hace rollback disponiendo cada entrada montada — incluyendo `hmr`, cuyo teardown drena sus tareas de refresh. La tarea de refresh disparada por el escaneo se sienta en la cola del Include detrás del mismo apply cuyo rollback está disponiendo a HMR: el rollback espera a HMR, HMR espera al refresh, el refresh espera al apply.

## Decisión

Ambas mitades se arreglan en los paquetes vendored (registrado en `vendor/README.md`):

- `include/src/index.ts` embudo cada mutación del árbol hijo — apply inicial, refresh y re-aplicación de parche `internal/update` — a través de una cola de promises por-Include. El `update` transaccional del grupo no es reentrante, así que la serialización es un requisito de corrección, no una elección de throughput. `refresh()` también lee dentro de la cola para que su chequeo de contenido-cambiado compare contra el estado comiteado del predecesor.
- `hmr/src/index.ts` pasa `ignoreInitial: true` al watcher principal. El escaneo inicial solo re-anuncia archivos que boot acaba de consumir; suprimirlo elimina tanto el refresh en tiempo de boot como los eventos `add` espurios para módulos ya cargados. `registerConfig()` conserva su propio watcher con `ignoreInitial: false` porque una config personal presente al registro debe aplicarse exactamente una vez.

Con ambos en su sitio, un boot fallido sigue la ruta pretendida: el apply único falla, el rollback dispone el árbol (corriendo el propio shutdown del TUI, restaurando el terminal), `loader.create` rechaza, y `boot()` relanza el diagnóstico etiquetado con exit 1.

## Alternativas consideradas

**Solo `ignoreInitial: true`.** Elimina el disparador pero deja la corrupción: cualquier refresh genuinamente concurrente (una edición de config corriendo carrera contra un apply lento) sigue intercalando dos updates de grupo y varando la fiber.

**Solo serialización.** Convierte la corrupción en el interbloqueo de rollback descrito arriba; el proceso sigue saliendo 13 en silencio.

**Cancelar los refresh encolados en el teardown de HMR.** Exige canalización de cancelación a través del bucle de tareas de `refreshConfig` y la cola del Include para un caso que `ignoreInitial` ya elimina de todo boot; no vale la maquinaria hasta que quede un disparador real.

## Consecuencias

Una edición del archivo de config que aterrice dentro de la ventana del escaneo de arranque del watcher la recoge ahora el siguiente evento `change` y no el escaneo mismo; el comportamiento de reload en estado estable no cambia.

Queda un hueco latente: una edición de config hecha durante un apply inicial *fallido* puede aún encolar un refresh que el teardown de HMR del rollback espera — la misma forma de interbloqueo con una ventana de disparo a escala humana de un boot fallido. Si algún día muerde, el fix es la cancelación de trabajos de refresh en el teardown de HMR.

## Testing

El caso PTY de proveedor inválido de `dsh` en `apps/cli/tests/tui-keyless-smoke.e2e.ts` fija el contrato extremo-a-extremo: exit 1, el diagnóstico etiquetado `dsh: plugin tree failed to load:` nombrando `$.providers`, y el reset de bracketed-paste probando que el árbol se disponió. Antes de este fix el mismo caso observaba exit 13 sin diagnóstico. El comportamiento de reload sigue cubierto por `packages/boot/app-boot/tests/config-reload.spec.ts` y `packages/boot/app-boot/tests/hmr-config.spec.ts`.
