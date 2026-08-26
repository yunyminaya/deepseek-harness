# Agent Note: Servidor de fallos de cable LLM scriptable

Status: implemented

[English](2026-07-25-scriptable-llm-wire-fault-server.md) | Español

## Problema

Los tests unitarios de adaptadores usan servidores HTTP locales para clasificar fallos individuales de provider, mientras que los tests de reintento usan un `LlmAdapter` scriptado en proceso para probar la recuperación de pasos cerrados. Ninguno de los dos límites proporciona un servidor reutilizable para correr juntos el adaptador HTTP de envío, el agent loop y la política de reintento, y ninguno permite que un desarrollador apunte una app existente a fallos de transporte deterministas cambiando solo su base URL y su clave de API.

El rechazo de conexión, un reset antes del primer evento, EOF limpio sin `[DONE]`, una finalización válida sin contenido, y un reset después de salida parcial tienen resultados distintos de adaptador y de recuperación. Tratarlos como un único fallo de mock genérico oculta si el límite del provider preservó la distinción y si los chunks fallidos permanecieron fuera de la historia de modelo confirmada.

## Decisión

`@deepseek-ai/dsh-llm-mock-server` es un paquete de soporte privado con un servidor HTTP de Node importable. La entrada de source `pnpm run mock:llm` local al repo proporciona un proceso independiente para inyección manual de fallos; el paquete no expone binario instalable. Acepta raíces compatibles con OpenAI y rutas de chat-completions `/v1`, valida un bearer token opcional, captura las peticiones y consume un comportamiento explícito por petición aceptada. El agotamiento de scripts falla en voz alta; la repetición requiere `repeatLast`.

Los comportamientos de petición cubren reset de socket, desconexión tras cabecera, desconexión parcial, stall, finalización vacía válida, streams truncados limpios, payloads malformados, fallos HTTP representativos, respuestas completas de texto/razonamiento/llamada de herramienta, streaming lento y finalización por max-token. Un `connection_refused` verdadero es una fase del ciclo de vida del listener de CLI porque un handler de petición vinculado no puede rechazar su propia conexión TCP.

La entrada de script `random` realiza una nueva selección ponderada por cada petición. El servidor expone y registra su seed de 32 bits sin signo, acepta pesos relativos suministrados por el llamante y envía un perfil de estrés con sesgo de éxito que mezcla resultados de transporte, protocolo, provider, timeout y semánticamente-vacíos. El perfil es presión de prueba configurable, no una estimación de la frecuencia de incidencias de producción; `connection_refused` permanece fuera del pool a nivel de petición.

El servidor reporta solo hechos de cable y no clasifica la reintentabilidad. Los tests de composición real lo enrutan a través de `dsh-llm-deepseek`, `dsh-agent-loop` y `dsh-llm-retry`: el rechazo de conexión, la desconexión dura, el reset parcial, el timeout de inactividad y una finalización válida sin contenido se recuperan bajo la política por defecto existente; el EOF parcial limpio permanece `STREAM_CLOSED` y no se reintenta por defecto. El paquete no cambia esas políticas.

## Verificación

Los tests del paquete ejercitan cada comportamiento de petición, la decodificación de peticiones UTF-8 partidas, la validación HTTP sin consumo de script, el agotamiento/repetición de scripts, el teardown de conexiones detenidas, el parseo de CLI y los límites de retardo, las base URLs IPv6, la reproducibilidad de la seed aleatoria, la validación de pesos, la telemetría de resultado único, la limpieza del ciclo de vida y el compañero invariante bajo la puerta de cobertura por archivo. La suite de integración de reintento prueba recuentos exactos de peticiones, pasos de reintento numerados, identidad del cuerpo de petición, aislamiento de chunks parciales fallidos, recuperación semánticamente-vacía, clasificación de EOF limpio, recuperación por timeout y recuperación real de conexión rechazada tras el arranque retardado del listener, y agotamiento acotado a través del adaptador HTTP/SSE real.

## Alternativas consideradas

**Implementar el servidor en Python** — rechazado porque las APIs HTTP y de socket estándar de Node exponen cada fallo requerido, mientras que TypeScript mantiene el servidor, el parser de CLI, los tests, el build del paquete, el lint y la cobertura dentro del toolchain existente del repositorio. Un segundo runtime añadiría dependencias de entorno y de subprocesos sin aumentar el aislamiento de cable.

**Mantener mocks de servidor inline separados en los tests de adaptadores** — rechazado porque esos fixtures no pueden ser lanzados por una app existente y duplicarían la secuenciación de comportamiento, la aleatorización, la telemetría y la limpieza de conexiones entre suites. Un paquete de soporte da a los tests una implementación compartida sin promoverla a API de producto.

**Usar solo un mock `LlmAdapter` en proceso** — rechazado porque elude fetch, el parseo de estado/cabeceras HTTP, el enmarcado SSE, la terminación de socket y el watchdog de inactividad del adaptador: los límites exactos que esta infraestructura de test existe para ejercitar.

**Exponer un binario de workspace instalable** — rechazado porque pnpm enlaza los binarios de dependencias antes de que existan los outputs de build del repositorio, acoplando instalaciones limpias a un artefacto solo-de-test. El comando de source local al repo soporta la misma inyección manual de fallos sin añadir una superficie de instalación de paquetes.

**Cambiar los valores por defecto de reintento con el servidor** — rechazado porque el servidor revela la semántica existente en lugar de decidir política. Extender la recuperación a `STREAM_CLOSED` requiere una decisión separada con sus propios trade-offs de costo, latencia y generación duplicada.

## Consecuencias

Los desarrolladores pueden reproducir secuencias de fallos cambiando solo la configuración de URL/clave del provider, y los tests automatizados pueden mantener deterministas los fallos a nivel de socket mediante scripts y seeds explícitos. El mismo fixture de cable ahora expone las brechas entre resets duros, truncado limpio y finalizaciones vacías recuperadas sin intentos de empalme ni modificación de la historia del modelo.

El servidor añade un paquete de soporte privado y un vocabulario de comportamiento que debe permanecer compatible tanto con los tests directos como con los ejemplos de CLI locales al repo. Los scripts ordenados por llegada se comparten intencionadamente entre clientes, los valores por defecto aleatorios son pesos de estrés más que verdad operacional, y el rechazo exacto de conexión requiere coordinar el intento del cliente con el intervalo previo al listen.
