# `@deepseek-ai/dsh-llm-mock-server`

[English](README.md) | Español

Un servidor HTTP/SSE compatible con OpenAI y programable para ejercitar adaptadores LLM reales, el agent loop y la política de recuperación sin clave de provider. Acepta `POST /chat/completions` y `POST /v1/chat/completions`; cada petición aceptada consume un comportamiento configurado en orden de llegada. Los métodos, rutas, tokens bearer y JSON inválidos no consumen el script.

El entry de la librería exporta `startMockLlmServer(options)`, los tipos de comportamiento y telemetría, los pesos de estrés aleatorios por defecto, el límite aceptado de temporizadores de Node y un handle en ejecución con la `baseURL` ligada, el `randomSeed` generado o configurado, las peticiones capturadas y un `close()` idempotente. El cierre termina a la fuerza las conexiones estancadas.

## Uso independiente

Ejecuta el entry de fuente desde este repositorio:

```sh
pnpm run mock:llm -- \
  --port 8000 \
  --api-key mock-key \
  --sequence partial_disconnect,success \
  --partial-text "discard this half"
```

Apunta el adaptador DeepSeek distribuido al servidor; este añade `/chat/completions` a la base configurada:

```sh
DEEPSEEK_BASE_URL=http://127.0.0.1:8000/v1 \
DEEPSEEK_API_KEY=mock-key \
pnpm dsh --profile headless "test provider recovery"
```

El script del repositorio escribe JSONL en stdout: un registro `ready` lleva la URL base `/v1` y la semilla aleatoria, seguido de registros de petición/resultado que nombran tanto el comportamiento programado como el comportamiento concreto seleccionado. El paquete de soporte privado no expone ningún binario instalable.

## Script de comportamiento

`--sequence` es un FIFO separado por comas. El agotamiento devuelve un HTTP 500 estructurado; `--repeat-last` reutiliza explícitamente la última entrada.

| Comportamiento | Resultado en la línea |
|---|---|
| `connection_reset` | Destruye el socket antes de las cabeceras HTTP |
| `stream_disconnect` | Envía las cabeceras SSE y luego reinicia antes del primer evento |
| `partial_disconnect` | Envía deltas de texto y luego reinicia el socket |
| `stall` | Envía las cabeceras SSE y permanece inactivo hasta la cancelación del cliente/servidor |
| `empty` | Envía una parada válida sin contenido y `[DONE]` |
| `empty_body` / `stream_eof` / `partial_eof` | Termina limpiamente sin el límite `[DONE]` requerido |
| `malformed_json` / `malformed_event` | Envía JSON SSE inválido o una forma de chunk de provider inválida |
| `rate_limit` / `server_error` / `service_unavailable` | Devuelve errores JSON 429/500/503 orientados a reintento |
| `auth_error` / `invalid_request` / `context_overflow` / `quota_exceeded` | Devuelve errores de provider terminales o recuperables por separado |
| `success` / `slow_success` / `reasoning_success` | Transmite una respuesta de texto completa, opcionalmente retrasada o precedida de razonamiento |
| `tool_call_success` / `max_tokens` | Completa con una llamada de herramienta o una finalización `length` |
| `wrong_content_type` | Envía un cuerpo SSE válido bajo `application/json` |
| `random` | Selecciona un comportamiento de petición concreto a partir de aleatoriedad ponderada con semilla |

`connection_refused` es solo de CLI y debe ser la primera entrada. Retrasa la vinculación de un puerto distinto de cero especificado por quien llama, de modo que las peticiones durante `--listen-delay-ms` reciben una negativa TCP real; las entradas restantes comienzan después de que el listener arranque.

## Modo aleatorio

Usa una entrada `random` repetida para una ejecución mixta abierta:

```sh
pnpm run mock:llm -- \
  --port 8000 \
  --sequence random \
  --repeat-last \
  --seed 42 \
  --random-weights 'success=60,slow_success=10,connection_reset=5,stream_disconnect=5,partial_disconnect=10,empty=5,server_error=5'
```

Omitir `--seed` genera una y la imprime en el registro `ready`. `--random-weights` acepta entradas relativas no negativas `behavior=weight` y exige al menos un comportamiento concreto positivo. El valor por defecto exportado es un perfil de estrés cargado de éxitos que contiene reinicio, desconexión, salida parcial, finalización vacía, estancamiento, 429/5xx, truncamiento limpio y JSON malformado; es presión de prueba, no una estimación de la frecuencia de incidentes de producción. `connection_refused` queda excluido porque un manejador de peticiones vinculado no puede producir una negativa real.

Cuando los pesos aleatorios incluyen `stall`, configura el cliente bajo prueba con un tiempo de espera corto de inactividad de stream para que el escenario termine con prontitud.

## Controles de temporización y contenido

El CLI expone `--success-text`, `--partial-text`, `--reasoning-text`, `--chunk-size`, `--chunk-delay-ms`, `--disconnect-delay-ms`, `--retry-after-ms`, `--request-id`, `--tool-name` y `--tool-arguments`. Los retrasos en milisegundos son enteros acotados dentro del rango de temporizadores de Node; `retryAfterMs` también debe ser positivo. La librería acepta las mismas opciones en camelCase. Un `apiKey` exacto opcional valida `Authorization: Bearer *** la omisión acepta cualquier token.

## Model Experience

Ninguna, ya que este servidor de pruebas sustituye el comportamiento de línea del provider sin invocar un modelo real.

#### Efecto de caché KV

Ninguno; las peticiones terminan localmente y nunca llegan a una caché de provider.

## Limitaciones conocidas y trabajo diferido

- **Los pesos aleatorios modelan presión de prueba, no incidencia de producción** — quienes llaman y quieren una distribución específica del entorno deben proporcionar pesos medidos y registrar la semilla emitida.
- **Los scripts de petición se ordenan por llegada** — los llamadores concurrentes comparten un solo cursor, así que la asignación determinista de fallos por sesión requiere instancias de servidor separadas.
- **La negativa de conexión real es una fase del ciclo de vida del listener** — el retraso del CLI debe solaparse con el intento del cliente; la selección aleatoria a nivel de petición solo puede reiniciar una conexión aceptada.
