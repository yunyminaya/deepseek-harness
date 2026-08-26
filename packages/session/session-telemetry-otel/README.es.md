# @deepseek-ai/dsh-session-telemetry-otel

[English](README.md) | [中文](README.zh.md) | Español

El backend OpenTelemetry para [el seam de telemetría](../session-telemetry/) — el único punto de entrada que carga un despliegue. Su `mode` decide si el seam sigue los eventos de sesión en vivo, reproduce el log canónico solo en el feedback registrado, o mantiene la telemetría local. Los modos de carga componen el SDK JS de OTel tal cual (`LoggerProvider` → `BatchLogRecordProcessor` → exportador de logs OTLP/HTTP) y asignan cada registro entregado a `logger.emit()`, bajo dos ámbitos de instrumentación: los registros de ledger en `@deepseek-ai/dsh-session-sessionTelemetry-otel`, los registros operativos en `@deepseek-ai/dsh-session-sessionTelemetry-otel/ops`. La identidad de recurso contiene `service.name`/`service.version` de `APP_IDENTITY` de `dsh-llm` más el `user.id` anónimo de este paquete (`$DSH_HOME/.anonymous-user-id`, un UUID aleatorio creado en el primer uso y reiniciable borrando el archivo), transportado una vez por lote de exportación y no por registro.

## Config

```yaml
- id: sessionTelemetry-otel
  name: '@deepseek-ai/dsh-session-sessionTelemetry-otel'
  config:
    mode: FULL                # explicit opt-in; default: DISABLED
    shutdownTimeoutMillis: 3000 # optional; defaults to 3000
    exporter:                # passed verbatim to the SDK's OTLP/HTTP log exporter
      url: https://collector.example.com/v1/logs
      headers:
        authorization: !!js `Bearer ${process.env.OTLP_TOKEN}`
    processor: {}            # optional; passed verbatim to BatchLogRecordProcessor
```

| `mode` | Comportamiento |
|---|---|
| `FULL` | Cada registro proyectado, incluidos los registros operativos de ciclo de vida, se entrega al SDK de OTel inmediatamente. |
| `FEEDBACK_ONLY` | Cada `feedback/record` reproduce, proyecta y redacta el sufijo del log de sesión canónico a través de ese evento. Los registros posteriores esperan a otro evento de feedback y permanecen locales si ninguno llega. |
| `DISABLED` | Por defecto. No se construye ningún coordinador, provider, procesador ni exportador. Ningún registro de telemetría sale del proceso. Un `feedback/record` registra `session sessionTelemetry is DISABLED; nothing will be shared and this feedback remains local`; el evento permanece en el log de sesión local. |

La configuración TypeScript programática usa el enum exportado `SessionTelemetryMode` (`SessionTelemetryMode.FULL`, `SessionTelemetryMode.FEEDBACK_ONLY` o `SessionTelemetryMode.DISABLED`); los literales de cadena en bruto no son asignables. La configuración Cordis serializada sigue usando los valores de cadena mostrados arriba.

La autorización de carga es positiva y fail-closed. Un modo de construcción directa desconocido falla antes de que se lea la configuración de transporte. Solo `FULL` acepta llamadas directas a `ctx.sessionTelemetry.emit()`. `FEEDBACK_ONLY` da a su coordinador bajo demanda una capacidad de backend privada y trata solo el objeto `feedback/record` exacto ya almacenado en `session.events[event.seq]` como consentimiento; un valor de bus emitido de forma independiente se ignora. `DISABLED` nunca construye la canalización del SDK, ni siquiera cuando hay opciones de exporter presentes.

El servicio montado revela el modo resuelto a través de la propiedad `sharing` del [`SessionTelemetrySharingStatus`](../session-telemetry/README.es.md#the-sharing-disclosure) del seam (`full` / `feedback-only` / `disabled`), de modo que el acuse de `/feedback` pueda informar de si la sesión se comparte y cómo. La revelación se fija en el constructor y es independiente de la captura: incluso `DISABLED` revela `disabled`.

`exporter.url` es obligatorio en `FULL` y `FEEDBACK_ONLY`, no tiene valor por defecto y debe parsear como `http(s)`; es opcional y no se usa en `DISABLED`. En los modos de carga, `shutdownTimeoutMillis` es un plazo externo positivo y finito propiedad de DSH que por defecto vale 3000 ms, y un `processor.maxExportBatchSize` entero no positivo también falla en la carga del plugin porque el SDK lo acepta pero luego se cuelga en el apagado. Ambos bloques del SDK pasan íntegros: cada campo de `OTLPExporterNodeConfigBase` (`headers`, `timeoutMillis`, `compression`, `keepAlive`, …) llega al exporter, y el batching, la cadencia de exportación (`scheduledDelayMillis`), el reintento, los límites de cola y la política de pérdida bajo fallos sostenidos son comportamiento del SDK ajustado a través de `processor`. El backend no implementa `flush()`: el procesador por lotes es dueño del volcado ordinario. Durante el apagado, OTel espera a `exporter.forceFlush()` antes de la promesa de finalización acotada por `exportTimeoutMillis` del procesador; si esa promesa de transporte nunca se resuelve, este paquete abandona la espera en `shutdownTimeoutMillis`, registra el fallo de apagado contenido a través del coordinador y deja que el teardown de la aplicación continúe. El plazo no puede cancelar el transporte del SDK, así que los registros aún pendientes pueden perderse en la salida del proceso.

## Lo que sale de la máquina

En los modos de carga, los registros llevan el `event.data` completo tal como lo devuelve el waterfall (cascada de eventos) `sessionTelemetry/record` del seam — contenido de mensajes de usuario y assistant, argumentos y resultados de herramientas (salida de comandos, contenidos de archivos), el prompt de sistema completo y los schemas de herramientas (`request/header`), texto de todo, resúmenes de compactación, `stderrSummary` de hooks, texto de feedback y el `cwd` de la sesión (una ruta local). El seam no trae reglas de redacción: sin un listener de `sessionTelemetry/record` montado, esa es la copia capturada en bruto, así que un despliegue que exporte más allá de un límite de confianza monta sus propias reglas (ver [el README del seam](../session-telemetry/README.es.md#the-redact-waterfall)). `FULL` ejecuta la redacción en el momento del append; `FEEDBACK_ONLY` no retiene ninguna copia de telemetría y ejecuta las reglas montadas en ese momento cuando el feedback dispara la reproducción del log canónico. Las credenciales de provider nunca aparecen, en ningún caso: las API keys de los adaptadores son parámetros de constructor, no eventos de sesión, así que están estructuralmente ausentes del log y por tanto de la telemetría. `DISABLED` no construye la canalización del SDK ni entrega captura alguna a un backend.

## Mapeo de campos

Registro del seam → registro de log del SDK: `time` → `timestamp`/`observedTimestamp`; `severity` → `severityNumber`/`severityText` (INFO 9 / WARN 13 / ERROR 17); `body` → el cuerpo de log estructurado; `attributes` verbatim. Los receptores deduplican por `(session.id, event.seq)` y alertan por severidad. En `FULL`, también pueden detectar crashes por la ausencia del registro `shutdown`: el marcador se emite en la propia disposición de la sesión o en el teardown de la aplicación, y un marcador seguido de más eventos es una recarga de telemetría. En `FEEDBACK_ONLY`, un prefijo liberado normalmente no tiene un marcador `shutdown` posterior, así que su ausencia no es una señal de crash. Los streams no son autocontenidos a través del linaje: una sesión reanudada continúa el stream de su propio id desde donde lo dejó el proceso anterior, y el stream de una sesión bifurcada arranca en su límite heredado — su prefijo vive en el stream del padre, cosido vía `session.parent_id` + `session.seed_length`. Un log local reanudado puede contener cierres sintéticos que nunca se exportaron; el stream del wire sigue siendo fiel a los registros realmente entregados al SDK.

## Experiencia del modelo

Ninguna, ya que el backend solo reenvía los registros redactados del seam a la canalización del SDK de OTel; nunca contribuye a una solicitud de modelo.

#### Efecto de KV Cache

Ninguno; este paquete ni ensambla ni envía una solicitud de provider.

## Limitaciones conocidas y trabajo diferido

- **Árbol experimental upstream** — `@opentelemetry/sdk-logs` sigue publicándose desde el árbol experimental upstream; el vaivén de la API del SDK aterriza aquí y solo aquí — el contrato del seam no se mueve.
- **El comportamiento del colector en vivo pertenece al exporter del SDK** — la autenticación, TLS, la limitación de velocidad y el resto del comportamiento real de despliegue OTLP siguen al SDK upstream en lugar de a una capa de compatibilidad propiedad del paquete.
- **Instantánea en el momento del feedback** — `FEEDBACK_ONLY` no retiene ninguna copia propiedad de la telemetría antes del feedback. Lee y redacta el log canónico actual cuando se registra el feedback; un crash antes del feedback no sube nada, y los cambios de política antes del feedback afectan a lo que esa reproducción exporta.
