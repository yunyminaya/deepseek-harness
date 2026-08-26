# @deepseek-ai/dsh-acp

[English](README.md) | Español

Servidor [Agent Client Protocol](https://agentclientprotocol.com) solo de automatización sobre JSON-RPC stdio. Los clientes programáticos crean agents nuevos del harness, envían prompts de texto/imagen, recogen el texto y las imágenes confirmados del asistente, resuelven por política las solicitudes de permiso de un solo uso y cancelan el trabajo. El cliente principal dentro del repositorio es [`dsh-subagent-acp`](../../subagent/subagent-acp/README.es.md).

Este paquete es un adaptador de transporte, no una integración de UI ni un seam de capacidad. No expone navegación de editor, replay de transcript, comandos, modos, selectores de configuración, elicitación, razonamiento, planes, títulos ni presentación de herramientas. El renderizado interactivo y las preguntas humanas pertenecen a los módulos host y cliente web.

## Plugin

`apply(ctx, config)` abre una `AgentSideConnection` en stdin/stdout y conduce `ctx.agents`. Stdout está reservado para los frames del protocolo.

| Config | Default | Significado |
|---|---|---|
| `provider` | — | Ruta de provider inicial para cada agent creado. |
| `model` | — | Modelo inicial para cada agent creado. |

Ambos campos son opcionales para que otro listener de agent/solicitudes pueda aportar el destino. La composición ACP ejecutable exige ambos.

## Contrato de protocolo

| Método | Comportamiento |
|---|---|
| `initialize` | Negocia la versión admitida. Los prompts de imagen solo se anuncian cuando hay montado un almacén duradero de adjuntos y el provider/modelo exacto configurado resuelve con entrada de imagen explícita; el audio y el contexto incrustado permanecen en falso. No se anuncia ninguna capacidad de sesión, editor, terminal, sistema de archivos ni MCP. |
| `authenticate` | No-op porque el servidor no anuncia ningún método de autenticación. |
| `session/new` | Crea un agent nuevo con un `cwd` primario absoluto; los `additionalDirectories` y `mcpServers` vacíos se aceptan, los valores no vacíos se rechazan. |
| `session/prompt` | Conserva el texto ordenado y los bloques de imagen en línea admitidos, renderiza los enlaces de recursos como referencias textuales entre corchetes y rechaza el audio, los recursos incrustados, la entrada malformada o vacía, o una imagen cuando la capacidad no se anunció. Valida todo el lote de imágenes y vuelve a comprobar la ruta exacta más reciente de la sesión antes de cualquier guardado, confirma cada imagen antes del evento de usuario, permite una solicitud en vuelo por sesión y espera la admisión más, una vez en cola, la inactividad de todo el Agent y la entrega ordenada de salida. La quietud normal informa `end_turn`; la cancelación ACP explícita, la disposición o un prompt cuya admisión se descartó (una ranura sin turno) informan `cancelled`. |
| `session/cancel` | Marca y aborta cualquier admisión en curso sin cancelar ni esperar trabajo del Agent no relacionado; una vez que este prompt ha entrado en la bandeja de entrada del Agent, cancela el Agent destinatario y espera a que el intervalo propio alcance la quietud. No se publica ningún mensaje de usuario tardío y el prompt se liquida como `cancelled`. Sin un prompt en vuelo, cancela el trabajo autónomo; los ids desconocidos son no-ops. |
| `session/update` | Emite un `agent_message_chunk` por cada bloque de texto o imagen no vacío en un `assistant/message` confirmado, conservando el orden. Las imágenes se releen y se verifica su integridad antes de la entrega base64 en línea. Los deltas brutos y los eventos que no son mensajes se omiten. |
| `session/request_permission` | Ofrece opciones de permitir/rechazar de un solo uso para las solicitudes de aprobación propiedad del bridge que llevan un id de llamada de herramienta. Los clientes pueden responder automáticamente. |

Una conexión puede ser dueña de varias sesiones. El bridge indexa los registros por id de sesión etiquetado y comprueba la identidad exacta del agent antes de enrutar eventos o solicitudes de permiso. Cada sesión tiene una ranura de prompt independiente, un espacio de trabajo, un camino de cancelación y un disposer.

La salida de mensajes confirmados cambia deliberadamente la latencia token a token por un resultado de automatización limpio. Los trozos de provider no confirmados y los intentos de reintento no pueden filtrar texto o imágenes parciales; el razonamiento y la actividad de herramientas permanecen en el log de la sesión para su observación a través de otras interfaces. La entrega por sesión se serializa porque las lecturas de adjuntos son asíncronas, y una imagen confirmada ausente o corrupta hace fallar la respuesta al prompt en lugar de emitir un marcador de posición.

## Ciclo de vida

La desconexión del cliente y la disposición de Cordis comparten un teardown memoizado. El bridge primero rechaza sesiones y prompts nuevos, cancela y lleva a la quietud la admisión de prompts, la actividad del agent y la entrega ordenada de salida, y luego drena los descendientes continuables solo por debajo de los Agents exactos propiedad de esta conexión antes de disponer esos handles en paralelo y esperar cada resultado antes de informar de cualquier fallo. Otros frontends que comparten el Context conservan sus bosques continuables y su admisión. Por tanto, una recarga de plugin solo ACP no deja ningún agent huérfano.

ACP exige que cada respuesta a un prompt lleve un `stopReason`, pero el bridge no reclama un resultado de turno específico del prompt. El intervalo de la operación empieza cuando el prompt entra en la bandeja de entrada del Agent y termina después de que la admisión, la inactividad de todo el Agent y la entrega ordenada de salida alcancen todas la quietud; los fallos de trabajo del Agent no relacionado anteriores a esa recepción en la bandeja no se atribuyen al prompt. Los mensajes confirmados del asistente se transmiten a lo largo del intervalo propio, y el steering o el trabajo inyectado pueden contribuir antes de la inactividad. La precedencia de liquidación es: cancelación explícita, fallo de entrega de salida, fallo del Agent en todo el intervalo y, después, el final del turno correlacionado. Los finales por límite de tokens se liquidan como `end_turn`; un error de modelo correlacionado solo rechaza en el mismo límite de quietud.

## Ejecución

`pnpm --dir /path/to/deepseek-harness run demo:acp` arranca la composición de servidor de automatización del repositorio. Un harness padre puede hacerle spawn a través de [`@deepseek-ai/dsh-subagent-acp`](../../subagent/subagent-acp/README.es.md); el resto de clientes ACP solo necesitan los métodos principales de arriba.

## Model Experience

### Texto e imágenes del prompt

#### Lo que ve el modelo

`session/prompt` conserva el orden texto/imagen en un único mensaje de usuario; el texto adyacente se concatena, y un enlace de recurso aparece como referencia entre corchetes `[resource_link name=… uri=…]` que el modelo puede abrir con sus propias herramientas. El base64 de las imágenes en línea se descarta tras la admisión del lote, de modo que el mensaje duradero contiene solo referencias de adjuntos verificadas. Los metadatos del protocolo, las capacidades del cliente, las opciones de permiso y los ids de sesión nunca entran en la solicitud al modelo.

#### Efecto de tokens

Los tokens del prompt y los cargos de imagen dependen de los datos y permanecen en el historial de esa sesión hasta la compactación. Las sesiones ACP concurrentes conservan contextos independientes.

#### Efecto de KV Cache

Solo de añadido; el nuevo mensaje de usuario sigue el prefijo de solicitud reutilizable y no invalida las entradas de caché anteriores.

### Decisiones de permiso

#### Lo que ve el modelo

Nada directamente. La herramienta propietaria registra su resultado de permitido, rechazado, cancelado o no disponible a través del camino normal de resultado de herramienta.

#### Efecto de tokens

Solo el resultado de la herramienta propietaria contribuye tokens.

#### Efecto de KV Cache

Solo de añadido a través del resultado de la herramienta propietaria.

## Limitaciones conocidas y trabajo diferido

- **Solo sesiones nuevas** — la carga, el listado, la reanudación, el borrado y el fork no están soportados.
- **Imágenes rasterizadas y un solo espacio de trabajo** — los prompts de imagen requieren un almacén duradero más una ruta exacta que declare entrada de imagen; solo se aceptan PNG, JPEG, WebP y GIF. El audio, los recursos incrustados, los directorios adicionales no vacíos y los servidores MCP se rechazan; los enlaces de recursos se aplanan a referencias textuales en lugar de contenido obtenido.
- **Solo respuestas confirmadas** — el progreso en vivo, el razonamiento, la actividad de herramientas, los planes, los títulos y el uso quedan fuera del cableado.
- **Vida ligada a la conexión** — una conexión libera todas sus sesiones; el cierre por sesión no está implementado.
