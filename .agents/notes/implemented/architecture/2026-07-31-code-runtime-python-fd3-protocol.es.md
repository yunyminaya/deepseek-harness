# Agent Note: el protocolo de frames fd-3 de code-runtime-python

Status: implemented

[English](2026-07-31-code-runtime-python-fd3-protocol.md) | Español

## Problema

El backend de code-runtime de CPython (`@deepseek-ai/dsh-code-runtime-python`, que llega a través de una pila de PRs) ejecuta cada programa del modelo en un subproceso `python3 -I` nuevo y puentea las llamadas de binding y los valores de completación por el fd 3 del hijo. Ese canal necesita un protocolo de wire en el que ambas partes coincidan, y el host no puede confiar en él: el código del modelo tiene acceso completo al fd 3 y puede forjar cualquier frame, de modo que todo frame entrante es entrada hostil que el host debe validar y reconstruir antes de leerlo. El protocolo también tiene que transportar JSON sin pérdidas sin el límite de profundidad que imponen `JSON.stringify`/`json.dumps`, porque el `CodeJsonValue` del seam no tiene límite de profundidad.

Esta capa de la pila entrega solo ese protocolo, de modo que la gran implementación de `PythonCodeRuntime` y su suite de integración con subprocesos reales aterrizan sobre un contrato de wire revisado en lugar de llegar fusionadas con él. La pila principal divide [#436](https://github.com/deepseek-harness/deepseek-harness/pull/436) —un PR único de 9000 líneas— en capas revisables; esta es la capa del protocolo, basada en la [extensión del seam](2026-07-31-code-runtime-portable-identifier-seam.es.md).

## Decisión

`src/protocol.ts` es el lado del host del vocabulario de wire y de su codec de frames hostiles:

- **`validateChildFrame`** valida la forma y RECONSTRUYE cada frame entrante. La unión de tiempo de compilación no significa nada en el fd 3 —un frame forjado puede llevar `null`, campos envenenados u omitir los obligatorios—, de modo que cada frame aceptado se reconstruye campo por campo: los extras forjados nunca viajan con él, un id de llamada no finito jamás puede repetirse en una respuesta, y la basura devuelve `undefined` para descartarse en lugar de lanzar una excepción en el manejador de mensajes del host.
- **`encodeJsonPlain` / `checkDoneValue` / `hasUnsafeIntegerToken` / `hasNonLosslessNumber`** son el codec JSON sin pérdidas y sus medidores. Recorren de forma iterativa (una pila explícita, no recursión), de modo que un valor profundo por debajo del presupuesto de bytes cruza intacto; `checkDoneValue` pliega la medición de bytes y la ausencia de pérdida numérica en un único recorrido que rechaza una carga útil que excede el presupuesto antes del trabajo INCREMENTAL que de otro modo añadiría —los hijos encolados—; las cadenas y las claves se miden con un escaneo de tamaño escapado sin asignación (`jsonStringBytesUpTo`), de modo que la copia escapada nunca se materializa. No vuelve a limitar la anchura del propio frame: `done.value` ya está `JSON.parse`-ado cuando se ejecuta la comprobación, de modo que el tamaño de la carga útil se paga aguas arriba y queda topado allí por el buffer de recepción fijo del fd-3 del host (una capa posterior de la pila), no aquí. Los dobles integrales fuera del rango seguro se serializan a través de dígitos de `BigInt`, de modo que cruza el entero exacto, no la forma redondeada de `String()`.
- **`logTruncationMarker`** produce el texto del marcador in-band que un ledger de logs emite cuando agota su presupuesto de bytes.

`py/protocol.py` refleja las formas de los mensajes como `TypedDict`s y redeclara las dos superficies contra las que EJECUTAN ambas partes —`PROTOCOL_FD = 3` y `log_truncation_marker`—, con texto idéntico byte a byte.

El esqueleto del paquete (`package.json`, `tsconfig.json`, `tsdown.config.ts`, `src/index.ts`, `src/invariant.ts`, el triplete del README) se publica aquí y no en una capa posterior de la pila: `check-workspace-constraints` lee incondicionalmente el package.json de cada `packages/<group>/<pkg>`, y los gates de cobertura y de topología de invariantes exigen que el paquete exista y compile en el momento en que existe su directorio. El PR posterior de backend-core extiende `src/index.ts` con `PythonCodeRuntime` y amplía las dependencias de `package.json`; como se basa en esta rama, se trata de ediciones, no de conflictos.

## Contrato de wire

Los frames son JSON-lines en el fd 3, un objeto por línea, dejando stdout/stderr libres para la salida propia del programa. Hijo → host: `boot-ack`, `call`, `log`, `done`. Host → hijo: `boot` (primer frame), `run` (tras `boot-ack`) y un `reply` por cada `call`. La bandera `truncated` del frame `log` marca el frame que ES el marcador de truncamiento del propio ledger del hijo, de modo que el host deja de capturar en el mismo punto que el hijo en lugar de inferirlo de su propio presupuesto. `done.error.kind` es uno de `exception`, `invalid-output`, `output-limit`; los presupuestos de wall/CPU, las anulaciones y la muerte del sustrato se observan del lado del host, no se transportan como frames.

## Alineación del mirror

La revisión de la ronda 12 de #436 encontró `py/protocol.py` desactualizado frente a `src/protocol.ts` en tres declaraciones —a `LogMessage` le faltaba `truncated`, a `DoneMessage.error` le faltaba `kind` y a `Namespace` le faltaba el `errorClass` opcional. Este PR alinea las tres al levantar el archivo, de modo que el mirror desactualizado no se arrastra hacia delante. Para mantenerlo alineado, `tests/protocol-mirror.e2e.ts` lanza un `python3` real y afirma, contra `src/protocol.ts`: `PROTOCOL_FD` y `log_truncation_marker` (las dos superficies que ejecutan ambas partes) y el conjunto de campos de wire obligatorios/opcionales de cada `TypedDict` —de modo que un campo renombrado o eliminado, o que un lado haga opcional un campo que el otro exige (exactamente la deriva de la ronda 12), hace fallar el test. Los *tipos* de los campos no se comparan a través de la frontera de lenguajes; ese residuo queda en la revisión.

## Alternativas consideradas

**Mover el codec JSON de Python (`_encode_json_plain` / `_decode_json_plain`) a `py/protocol.py` por simetría entre lados con `protocol.ts`.** Rechazado. La regla del repositorio de «preferir simetría para valores paralelos» apunta a valores genuinamente paralelos; estos no lo son. El codec del lado del host en `protocol.ts` valida entrada HOSTIL y es autocontenido. El codec de Python produce salida en el lado CONFIABLE y está acoplado a helpers internos del bootstrap (`_Emit`, `_dump_scalar`/`_dump_string`/`_dump_float`, la contabilidad de costes de `LogBuffer`, `_check_done_value`, `_lossless_json_violation`); levantar solo los dos puntos de entrada arrastraría esa red a `protocol.py` o crearía un ciclo de importación `bootstrap.py` ↔ `protocol.py`. El paralelismo real entre lados es «el host valida lo entrante (`protocol.ts`) ↔ el hijo confía en el host y emite (`bootstrap.py`)», y esa simetría se conserva: `protocol.py` sigue siendo el mirror puro del vocabulario de wire que es en el lado TS. El codec de Python se queda en `bootstrap.py`, entregado por el PR de backend-core.

**Aplazar el esqueleto del paquete al PR de backend-core que «posee» package.json.** Rechazado: los gates de restricciones de workspace, cobertura y topología de invariantes fallan en el instante en que existe el directorio `code-runtime-python` sin un paquete compilable. Una división en pila no puede crear archivos fuente en un paquete que todavía no compila.

## Consecuencias

Comprado: el protocolo fd-3 y su codec de entrada hostil aterrizan como una capa autocontenida con cobertura unitaria completa, y la deriva del mirror py/ts que encontró la revisión de la ronda 12 queda corregida con una guardia en ejecución contra su reaparición. El PR de backend-core se construye sobre un contrato de wire revisado.

Coste: `src/index.ts` y `package.json` se introducen aquí de forma mínima y el PR de backend-core los edita (no los crea). El e2e del mirror compara los NOMBRES de los campos y su obligatoriedad/opcionalidad entre los dos lados, pero no los TIPOS de los campos —comparar declaraciones de tipos entre TypeScript y Python no tiene equivalente mecánico, de modo que ese residuo queda en la revisión más la suite de subprocesos reales del backend.
