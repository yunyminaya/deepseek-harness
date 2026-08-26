# @deepseek-ai/dsh-sdk-jsonrpc-demo

[English](README.md) | Español

Aplicación solo de bin que arranca un `cordis.yml` externo; su entrada [`jsonrpc`](../../sdk/server/README.es.md) atiende a clientes del SDK a través de stdio delimitado por saltos de línea. La configuración compone el spine, los backends y el plugin de servicio. El bin publicado `dsh-jsonrpc-agent` resuelve los plugins desnudos desde el proyecto de configuración. El [runtime de ejecutable único](../../../.agents/notes/implemented/architecture/2026-07-10-single-file-executable-sdk-runtime-distribution.md) `dsh-jsonrpc-agent-pkg` del SDK de Python usa `lib/packaged-bin.js` en su lugar: los plugins desnudos empaquetados se resuelven desde su árbol de runtime cerrado, mientras que los plugins relativos siguen siendo relativos a la configuración.

## Descubrimiento de configuración

Gana el primer canal no vacío: `$DSH_CORDIS_CONFIG` y luego `argv[2]` posicional. Si ninguno nombra un archivo existente, el bin imprime una línea de uso en stderr y sale con 1; no hay respaldo de directorio de trabajo ni integrado. [`dsh-app-boot`](../../boot/app-boot/README.es.md) hace fatales los fallos de carga de plugins. Este protocolo no usa `DSH_SNAPSHOT`.

Una configuración sin `dsh-sdk-jsonrpc-server` es válida y no sirve nada; el bin no designa un plugin de servidor.

## Ciclo de vida de salida

El EOF de stdin y `SIGTERM` liberan la raíz hasta la quiescencia y salen con 0; `SIGINT` sale con 130 tras la misma liberación. El EOF puede cortar un turno en curso, como documenta la [Agent Note de distribución](../../../.agents/notes/implemented/architecture/2026-07-10-single-file-executable-sdk-runtime-distribution.md). El plugin `jsonrpc` es dueño del cierre del protocolo de respuesta antes de salir; ambas rutas son idempotentes y seguras frente a carreras.

## stdout es el protocolo

stdout transporta solo tramas JSON-RPC. El bin y las guardas de arranque diagnostican en stderr, y la configuración debe omitir los loggers de stdout.

## Experiencia del modelo

Indirectamente, a través de los plugins cargados desde el `cordis.yml` externo, que son dueños de cada prompt, schema, mensaje y resultado vinculados al modelo; este bin no añade ninguno propio.

#### Efecto de KV Cache

Sin invalidación directa; el consumidor nombrado es dueño de cualquier cambio de prefijo de petición.

## Limitaciones conocidas y trabajo aplazado

- **El bin no puede demostrar que la configuración sirva JSON-RPC** — una configuración válida sin entrada `dsh-sdk-jsonrpc-server` arranca con éxito y no sirve nada.
- **No existe configuración integrada ni por defecto** — cada lanzamiento debe aportar `DSH_CORDIS_CONFIG` o una ruta posicional, y el despliegue es dueño del árbol de plugins completo y de la disciplina de stdout.
- **El EOF de stdin corta el trabajo en curso** — la desaparición del cliente libera la raíz de inmediato; quienes necesiten una finalización ordenada usan la solicitud de cierre a nivel de protocolo.
