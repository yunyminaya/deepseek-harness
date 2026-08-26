# Agent Note: fail-loud libera el terminal antes de salir

Status: implemented

English | [中文](2026-07-31-fail-loud-releases-the-terminal.zh.md)

## Problema

Un lanzamiento de `dsh` cuya configuración falló la validación imprimía su diagnóstico y devolvía al usuario a un shell roto. Teclear era invisible, y el siguiente comando quedaba destrozado por texto suelto:

```
dsh: fatal load failure: ValidationError: invalid config:
  - $.providers expected object but got [object Object] (at providers)
$ 1;2;4cecho hello
zsh: command not found: 4cecho
```

El Loader monta entradas concurrentemente, así que el orden de fallo de entradas no es el orden de arranque. `ui-tui` se activa y llama al `ProcessTerminal.start()` de pi-tui, que pone stdin en raw mode, habilita bracketed paste y escribe la sonda del protocolo de teclado Kitty — una secuencia que termina en una consulta Device Attributes (`ESC [ c`). Una entrada hermana (aquí `llm-pi-ai`) luego rechaza por su propia config. En ese momento, ese rechazo surgía como un unhandled rejection, e `installFailLoud` escribía una línea en stderr y llamaba `process.exit(1)` de inmediato. (El Loader transaccional ahora liquida los fallos del árbol de config a través de `boot()`, que dispone el contexto parcial por sí mismo; el hook de release sigue siendo el guard para rechazos que `boot()` no puede ver — trabajo async desprendido de un plugin que rechaza durante o tras el montaje.)

Nada disponía el árbol, así que `ProcessTerminal.stop()` jamás corrió: raw mode, bracketed paste y el protocolo de teclado quedaron activos en el shell que sobrevivió al proceso. La respuesta del terminal a la consulta Device Attributes (`1;2;4c`) llegó tras el exit y fue leída por el shell como input tecleado — el texto literal de arriba.

La ruta `/exit` jamás estuvo afectada, porque dispone el árbol y alcanza el propio `shutdown()` del TUI, que llama `drainInput()` (absorbiendo la respuesta pendiente) y luego `ui.stop()`. El defecto era que un *boot fallido* no tenía ruta a ese mismo teardown.

## Decisión

`installFailLoud` toma un teardown `release` opcional, esperado entre el diagnóstico y el exit:

- El diagnóstico se escribe **antes** del release, así que un disposer colgado o fallido no puede tragarse la razón.
- Un latch, y no un uninstall, conserva el primer rechazo como el reportado. Quitar el listener durante el teardown dejaría que un segundo rechazo concurrente se volviera uncaught, y Node mataría el proceso a mitad del teardown — varando exactamente el estado del terminal que esto restaura. Los rechazos posteriores, incluido el propio del release, caen al exit pendiente.
- El release está acotado por `FAIL_LOUD_RELEASE_TIMEOUT_MS` (2s) y su rechazo se traga. Un disposer wedgeado o fallido retrasa el exit fatal; jamás lo cancela. Ese timer permanece **referenced**: uno con `unref()` deja a Node alcanzar un event loop vacío y salir 0 justo en el fallo que se reporta, porque un listener `unhandledRejection` suprime el exit fatal por defecto.
- Omitir `release` conserva exactamente el comportamiento previo, así que los bins ACP, JSON-RPC y demo quedan sin cambio.

El launcher TUI de `dsh` pasa un release que dispone el contexto raíz, que corre el `shutdown()` existente del TUI y devuelve el terminal.

El launcher captura el contexto raíz en el hook `prepare` de `boot()` y no desde su valor de retorno. El rechazo llega mientras `boot()` sigue en vuelo, así que un `app.current` asignado tras el `await` seguiría `undefined` exactamente en el momento en que el hook lo necesita. `prepare` corre tras la instalación del Loader y antes de que cualquier entrada del árbol de config monte, lo que cubre toda la ventana en que una entrada puede rechazar.

## Alternativas consideradas

**Resetear el terminal desde el handler fail-loud** (escribir `ESC [ ? 2004 l`, popear el protocolo de teclado, limpiar el raw mode). Esto duplica el teardown de pi-tui en un paquete que no posee terminal, y se desviaría conforme cambie la secuencia de arranque de pi-tui. Tampoco puede absorber la respuesta Device Attributes en vuelo, que es lo que corrompe el siguiente prompt — solo drenando stdin mientras sigue raw se logra.

**Registrar un reset de terminal en `process.on('exit')` desde el TUI.** Los handlers de exit son síncronos, así que no pueden esperar `drainInput()`; la respuesta suelta igual aterrizaría. Además pone el teardown en un hook global en vez de en la ruta de disposición que ya existe.

**Hacer que el TUI se niegue a arrancar hasta que el árbol se liquide.** Esto serializa un Loader deliberadamente concurrente y retrasa el primer paint de cada lanzamiento sano para arreglar una ruta de fallo.

**Reordenar las entradas de config para que `llm-pi-ai` monte antes que `ui-tui`.** El orden no es una garantía que el Loader dé, y cualquier entrada futura podría fallar tras el montaje del TUI.

## Consecuencias

Un boot fallido cuesta ahora una disposición de árbol (acotada a 2s) antes del exit, y el código de salida sigue en 1. A cambio, un `dsh` mal configurado devuelve un shell usable en vez de uno que necesita `stty sane` o `reset`.

La garantía pertenece a cualquier bin que posea el terminal: una superficie que agarre estado de terminal y no pase `release` reintroduce este defecto. `installFailLoud` no puede detectarlo solo, ya que no tiene vista de lo que un plugin montado le hizo al proceso.

## Testing

`packages/boot/app-boot/tests/app-boot.spec.ts` cubre el contrato del release: el hook se espera antes de que el exit se comita, un hook que rechaza igual sale 1, uno que nunca se liquida sale tras `FAIL_LOUD_RELEASE_TIMEOUT_MS`, y una ráfaga de rechazos reporta solo el primero mientras el release igualmente completa.

Esos tests de fake-process no pueden observar los dos modos de fallo que más importan — el código de exit del proceso con un event loop real, y el estado del terminal tras el exit — así que la regresión vive en `apps/cli/tests/tui-keyless-smoke.e2e.ts`. Arranca el árbol shipped en un PTY real sobre `fixtures/tui-invalid-provider.cordis.yml` (un `providers` con forma de lista, el error que los usuarios de verdad cometen), espera exit 1, y aserta que los bytes capturados contienen tanto el rechazo de boot etiquetado (`dsh: plugin tree failed to load:`) como `ESC[?2004l`. El mismo caso fija la ruta de boot extremo a extremo: atrapó el [interbloqueo de arranque HMR initial-scan](2026-08-03-hmr-initial-scan-boot-deadlock.es.md) que salía silenciosamente con 13 y el terminal varado.

La ruta `/exit` conserva su aserción existente de que el mismo reset aparece en un exit limpio.
