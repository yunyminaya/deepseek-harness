# Agent Note: Apagado por señal acotado y escalante para Web y headless

Status: implemented

English | [中文](2026-08-03-cli-signal-shutdown-escalation.zh.md)

## Problema

El mount de telemetría por defecto añadió handlers SIGINT/SIGTERM a `dsh web` y al comando headless (hoy `dsh --profile headless`) para que el exit del proceso pudiera drenar el árbol Cordis en vez de soltar telemetría encolada. Cada handler usaba un latch booleano de una vía y salía solo tras liquidarse `ctx.fiber.dispose()`. La finalización normal headless también esperaba esa disposición sin cota.

Un usuario reprodujo entonces el comando headless colgado inmediatamente tras la URL de observación e ignorando `Ctrl+C` repetidos; `DSH_TELEMETRY_DISABLED=1` quitaba el cuelgue, mientras un handler Node aislado en el mismo sandbox Linux recibía SIGINT. Eso aisló el disposer pendiente a telemetría y no al reenvío de señales de terminal. El `BatchLogRecordProcessor.shutdown()` de OTel espera `exporter.forceFlush()` antes de la promesa de completitud acotada por `exportTimeoutMillis`, y el `forceFlush()` del exportador OTLP espera directamente su Promise HTTP en vuelo. Una conexión de proxy/sandbox que jamás obtenga socket puede por tanto dejar el shutdown del proveedor pendiente a pesar de ambos timeouts configurados del SDK.

El latch convirtió entonces ese defecto de telemetría en un CLI inmatable: la finalización normal ya esperaba la disposición única de la raíz; el primer SIGINT se unía a esa misma disposición pendiente y ponía el latch de señal; los SIGINT posteriores retornaban en el latch, así que el proceso no tenía escape restante. Una señal recibida antes de la finalización normal tenía la misma espera sin cota. Web usaba la misma forma de latch.

Los timeouts del propio SessionTelemetryBackend no pueden probar que todo el árbol de plugins se liquide. Cualquier disposer presente o futuro puede wedgearse, y el límite del proceso debe conservar tanto un primer intento grácil como una salida controlada por el usuario.

## Decisión

El fix tiene dos capas de propiedad. El backend OTel añade `shutdownTimeoutMillis` (valor por defecto y shipped: tres segundos) alrededor de la Promise completa de shutdown del proveedor SDK. Cruzarlo rechaza hacia la ruta de fallo-contenido existente del coordinador de telemetría, permitiendo que el árbol Cordis termine su disposición; los registros pendientes pueden perderse porque OTel no expone cancelación para la Promise de transporte.

Web y headless comparten `createProcessShutdown`, un controlador a nivel de proceso alrededor de la disposición de la raíz:

- Las llamadas de shutdown normal coalescen sobre una disposición y retienen el primer código de exit pedido; jamás se escalan entre sí. Una disposición exitosa registra ese código vía `process.exitCode` y deja a Node drenar sus handles restantes de forma natural. El fallo de disposición igual fuerza el exit del proceso porque el launcher no puede asumir que el árbol fallido alcanzó quietud.
- La primera señal arranca la misma disposición grácil y un backstop de exit referenciado de cinco segundos. Éxito o fallo de la disposición sale una vez; ninguno puede cancelar el exit del proceso.
- Una señal recibida mientras el shutdown está pendiente fuerza exit inmediato con el código de esa ruta de señal. Esto incluye el primer `Ctrl+C` tras una finalización normal headless que ya entró en disposición, y una segunda señal tras una señal que inició el drenaje.
- La cota de cinco segundos es un invariante de seguridad del proceso, no un tunable de despliegue. Es lo bastante larga para el techo ordinario de drenaje del despliegue de telemetría mientras sigue acotando cualquier disposer wedgeeado en el límite del launcher.

La finalización normal evita deliberadamente `process.exit()`: un exit forzado inmediatamente tras una petición Undici puede golpear la [aserción de async-handle libuv de Windows en Node](https://github.com/nodejs/node/issues/56645) antes de que drene la limpieza de handle nativo de la petición completada. Una señal todavía puede forzar el exit tras completarse la disposición normal si otro handle mantiene vivo el proceso.

Headless preserva exit 0 para un turno completado, exit 1 para otra razón de fin-de-turno o error de negocio de API, 130 para SIGINT y 143 para SIGTERM. Web preserva su comportamiento existente de SIGTERM exit 0 y SIGINT exit 130.

Esto sustituye el supuesto de la [nota de despliegue de telemetría](../feature/2026-07-31-web-telemetry-default-mount.es.md) de que los timeouts de exportador/procesador del SDK acotaban el shutdown completo del proveedor, y su decisión previa de diferir un backstop a nivel de proceso. El backend posee su política de pérdida/latencia de exportación y cierra el hueco conocido del `forceFlush()` del SDK; el launcher posee la garantía exterior de que ningún plugin puede atrapar el proceso indefinidamente.

## Alternativas consideradas

**Acotar solo el `shutdown()` del backend de telemetría.** Insuficiente porque protege la espera OTel conocida pero no puede proteger al launcher del disposer de otro plugin.

**Restaurar el exit inmediato por defecto de Node ante señal.** Descartado porque una primera señal sana todavía debe flushear telemetría y liberar otros recursos. El exit inmediato es la ruta de escalación explícita, no el defecto.

**Añadir solo el timeout de cinco segundos.** Descartado porque un usuario que presiona `Ctrl+C` de nuevo está pidiendo dejar de esperar ya. Tragarse esa intención el resto del periodo de gracia recrea el comportamiento reportado a menor duración.

**Llamar siempre `process.exit()` tras una disposición exitosa.** Descartado porque la disposición de la raíz prueba que el árbol de aplicación está en quietud, no que Node y sus dependencias nativas terminaron de retirar cada handle asíncrono. Fijar `process.exitCode` preserva el estatus pedido mientras deja al runtime terminar ese trabajo.

## Consecuencias

Un exit normal sano sigue disponiendo el árbol Cordis completo y luego espera a que el event loop de Node drene. La espera de telemetría conocida se libera tras como mucho tres segundos; cualquier otro exit wedgeeado dura como mucho cinco segundos sin más input, y una señal termina de inmediato una finalización normal demorada o un shutdown pendiente. El exit forzado o acotado por deadline puede interrumpir la exportación de telemetría o limpieza restante, lo cual es intencional solo tras fallar el contrato grácil o cuando el usuario escaló explícitamente.

El controlador es infraestructura del launcher y no un plugin Cordis: no afirma que la disposición completó, y no debilita la regla de ciclo de vida de que los disposers ordinarios deben alcanzar quietud.

## Testing

`apps/cli/tests/process-shutdown.spec.ts` fija la finalización natural tras disposición resuelta, el exit forzado tras disposición rechazada, el backstop de cinco segundos, la coalescencia de llamadas normales, la disposición propiedad-de-señal, una señal interrumpiendo disposición normal o el drenaje de handles post-disposición, y la escalada de segunda señal.

`apps/cli/tests/headless-shutdown.e2e.ts` arranca el árbol Loader Web/headless real shipped en un PTY con un plugin solo-test cuyo disposer anuncia entrada y jamás se liquida. El test envía SIGINT tras la URL de observación, espera prueba de que la disposición arrancó, envía SIGINT de nuevo, y exige exit 130. El resolver de lanzamiento fuente/artefacto conserva la misma regresión en ambos planos de ejecución. Este caso PTY cubre el estado de proceso visible al usuario; ningún snapshot de salida de modelo cambia.

`packages/session/session-telemetry-otel/tests/otel.spec.ts` sostiene una petición OTLP real abierta tras arrancar la exportación por timer y fija que la disposición Cordis retorna en `shutdownTimeoutMillis`, a pesar de que el `forceFlush()` del SDK sigue pendiente. El collector se libera luego para que la Promise del proveedor aún observada se liquide limpiamente.
