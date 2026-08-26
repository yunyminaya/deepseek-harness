# @deepseek-ai/dsh-code-runtime-python

[English](README.md) | Español

Implementación por subproceso CPython del seam [`@deepseek-ai/dsh-code-runtime`](../code-runtime/README.es.md). Compañero de [`@deepseek-ai/dsh-code-runtime-worker-thread`](../code-runtime-worker-thread/README.es.md); cambia el hilo de trabajo de Node por un subproceso `python3` nuevo para que el código del modelo sea Python en lugar de TypeScript.

El paquete es dueño del protocolo de cable de ese seam: el codec de frames del lado del host y el espejo del lado de Python del mismo vocabulario de mensajes.

## Protocolo de cable

El host y el subproceso CPython intercambian un protocolo sin versión de líneas JSON por el fd 3 del hijo — un objeto JSON por línea, dejando stdout/stderr libres para la salida propia del programa. `src/protocol.ts` es el lado del host; `py/protocol.py` refleja sus formas de mensaje y el texto compartido del marcador de truncamiento en el lado de Python.

- **fd 3, no stdout** — Node fija el canal posicionalmente con `stdio: ['pipe','pipe','pipe','pipe']`; el bootstrap de Python lee la misma constante `PROTOCOL_FD`. Encuadre de líneas JSON.
- **El host trata cada frame entrante como hostil** — el código del modelo tiene acceso completo al fd 3 y puede publicar cualquier cosa a través de él, así que `validateChildFrame` valida la forma y RECONSTRUYE cada frame antes de que el host lo lea: los campos extra falsificados nunca viajan de polizón, un id de llamada que no sea número jamás puede repetirse en una respuesta, y la basura cae a `undefined` en lugar de lanzar una excepción en el manejador de mensajes del host. El lado de Python confía en las respuestas del host (el host no está controlado por el modelo).
- **Cruce JSON sin pérdida** — los valores de finalización y los argumentos de bindings cruzan como JSON exacto. `encodeJsonPlain` serializa sin recursión un valor producido por `JSON.parse`, de modo que un valor profundo por debajo del presupuesto de bytes cruza intacto en lugar de morir en el límite de pila de `JSON.stringify`; `checkDoneValue` mide la longitud de bytes Y la ausencia de pérdida numérica de un valor de finalización falsificado en un solo recorrido que rechaza un payload por encima del presupuesto antes del trabajo incremental que añadiría (los hijos encolados; las cadenas y claves se miden con un escaneo de tamaño escapado sin asignación, así que la copia escapada nunca se materializa) — el ancho propio del frame ya se parsea y limita aguas arriba en el buffer de recepción del fd 3 del host, no se vuelve a acotar aquí; `hasUnsafeIntegerToken` lee el texto crudo del frame para detectar un token entero que `JSON.parse` redondearía en silencio; `hasNonLosslessNumber` rechaza un número no finito o un cero negativo en `call.args` sin acotar. Los dobles integrales fuera del rango seguro se serializan a través de dígitos `BigInt` para que cruce el entero exacto, no la forma redondeada de `String()`.
- **Marcador de truncamiento compartido** — `logTruncationMarker(maxBytes)` produce texto idéntico byte a byte en ambos lados, de modo que una ejecución de log truncada se lee igual sin importar cómo se alcanzó el tope. La bandera `truncated` del frame `log` distingue el marcador propio del registro del hijo de la salida del programa.

## Experiencia del modelo

Indirectamente, a través de Code Mode en [`dsh-tools`](../../core/tools/README.es.md), que renderiza el valor de finalización exacto de este backend cuando cabe (o un fallo explícito `invalid-output` / `output-limit`), más el marcador de log exacto `[dsh-code-runtime-python] log capture truncated at <maxLogBytes> bytes`, dentro de un resultado `run_code` retenido.

#### Efecto en la caché KV

Sin invalidación directa; el consumidor nombrado es dueño de cualquier cambio en el prefijo de solicitud.

## Limitaciones conocidas y trabajo diferido

- **La guardia entre lenguajes cubre las superficies ejecutadas por el runtime y las formas de los campos de frame** — `tests/protocol-mirror.e2e.ts` lanza un `python3` real y verifica, contra `src/protocol.ts`, tanto `PROTOCOL_FD` / el texto del marcador de truncamiento de log como el conjunto de campos de cable obligatorios/opcionales de cada `TypedDict` en `py/protocol.py`. Lo que no compara son los *tipos* de los campos (p. ej. que `cpuSeconds` sea un `int` en ambos lados): comparar declaraciones de tipos entre TypeScript y Python no tiene equivalente mecánico aquí, así que una deriva a nivel de tipos la siguen atrapando la revisión más la suite de subproceso real del backend, no los tests de este paquete.
- **`src/index.ts` exporta solo el vocabulario del protocolo** — el paquete no lleva ninguna ruta de ejecución por subproceso ni codec JSON del lado de Python, así que nada aquí lanza `python3` fuera del test espejo.
