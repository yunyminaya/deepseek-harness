# @deepseek-ai/dsh-sdk-jsonrpc-server

[English](README.md) | Español

El plugin `jsonrpc` sirve JSON-RPC delimitado por nuevas líneas sobre stdio para que los clientes SDK fuera de proceso puedan controlar agents del harness. [`HarnessSdkJsonRpcServer`](src/server.ts) es el dueño de los métodos y notificaciones del protocolo; el transporte y los tipos de wire con nombre viven en [`dsh-sdk-protocol`](../protocol/README.md), compartidos con los SDK de cliente; [`jsonrpc-demo`](../../examples/jsonrpc-demo/README.md) aporta la aplicación `cordis.yml` circundante.

## Cableado

`inject: ['agents']`. El servidor obtiene o crea un agent por `sessionId`. Reenvía las completaciones de subagent solo cuando la bandera `local` del ciclo de vida en instantánea del servicio es verdadera; los nombres de provider, los ids de hijo y el linaje durable nunca establecen localidad. Un adaptador registrado gana, una ruta `deepseek-official` sin dueño monta `dsh-llm-deepseek`, y cualquier otro provider sin dueño falla la inicialización. El resto de capacidades vienen de la `cordis.yml` circundante.

## Config

`maxTokensAsSuccess` por defecto es `false` y afecta solo al estado mapeado al despliegue en `subagent.finished`; los prompts de root-session no tienen estado a nivel de prompt. `JsonRpcConfig.input`, `output` y `exit` son hooks de transporte solo de runtime; la producción usa el stdio del proceso y `process.exit`.

## stdout es el protocolo

Stdout lleva solo frames JSON-RPC. El despliegue no debe componer un logger de stdout; los diagnósticos pertenecen a stderr.

## Semántica de shutdown y salida

El plugin responde `shutdown`, hace flush de la respuesta, dispone el contexto raíz para que los agents propiedad del SDK, las suscripciones y la persistencia alcancen la quiescencia, y luego sale con código 0. La salida por EOF y por señal pertenece al bin de la app, que también dispone el contexto raíz. Descargar solo este plugin detiene el servicio sin sacar el proceso.

## Notas de wire

`initialize` es la frontera de disponibilidad del runtime: cuando el servidor lo monta una composición de Loader, espera a que el árbol de plugins actual se asiente antes de responder, así las capacidades hermanas asíncronas como el descubrimiento inicial de herramientas MCP son visibles para el primer prompt. Los contextos construidos a mano sin Loader siguen siendo utilizables de inmediato. `initialize.serverInfo.name` es el estable de wire `deepseek-harness-sdk-runtime`. Un `initialize.maxTokens` positivo opcional se convierte en el tope de salida de petición de cada agent creado por SDK y de sus descendientes en proceso; los valores inválidos rechazan la inicialización, mientras que omitirlo no envía ningún tope SDK y permite que se aplique el default del adaptador o la ruta de provider seleccionados. `session/prompt` encola un mensaje de usuario identificado y devuelve de inmediato `{ messageId }`. El servidor transmite cada hecho durable como `session.event` y cada transición de ciclo de vida de todo el agent como `session.status`; no asigna un mensaje de assistant ni `turn/end` a ese prompt. Las peticiones independientes pueden encolar más trabajo en la misma sesión. Las raíces de persistencia y la persona vienen de `cordis.yml`.

## Experiencia del modelo

### Mensaje de usuario del SDK

#### Lo que ve el modelo

Por cada `session/prompt` aceptado, el modelo de conversación recibe los `contentBlocks` aportados por el caller verbatim como un mensaje de usuario en esa sesión de SDK. Este paquete no añade prosa de system prompt ni schema de herramienta; esos vienen de los plugins de la `cordis.yml` circundante.

#### Efecto en tokens

Los tokens de mensaje de usuario dependientes de datos entran en el historial de sesión retenido y se reenvían en turnos posteriores hasta que otro paquete los compacte. Los frames JSON-RPC, las notificaciones de sesión y la contabilidad del servidor añaden cero tokens de contexto de modelo.

#### Efecto en la caché KV

Solo append; el contenido recién visible sigue el prefijo de petición reutilizable y no invalida las entradas existentes de la caché KV.

## Limitaciones conocidas y trabajo diferido

- **El wire no tiene método de cierre por sesión ni de cancelación de prompt** — los agents creados por SDK siguen en vivo hasta el shutdown del proceso.
- **No hay resultado por prompt** — `MessageId` identifica solo la admisión en la inbox; los clientes que son dueños de un intervalo de automatización deben definir y observar ese intervalo ellos mismos.
- **La pureza de stdout la impone el despliegue** — una config circundante puede cargar igualmente un logger de stdout y corromper el canal JSON-RPC; este plugin no inspecciona ni veta loggers hermanos.
- **El montaje automático de adaptadores es específico de DeepSeek** — `initialize` puede reutilizar cualquier adaptador de modelo pre-registrado, pero su único fallback monta `dsh-llm-deepseek`.
